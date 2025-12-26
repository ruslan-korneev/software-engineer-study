# GitHub CLI

## Что такое GitHub CLI (gh)

**GitHub CLI (gh)** — это официальный инструмент командной строки от GitHub, который позволяет работать с GitHub напрямую из терминала. Вместо того чтобы переключаться между терминалом и браузером, вы можете выполнять большинство операций GitHub прямо в командной строке.

### Ключевые возможности:
- Создание и управление репозиториями
- Работа с issues и pull requests
- Просмотр и запуск GitHub Actions
- Управление релизами и gists
- Интеграция с Git-командами

---

## Установка

### macOS

```bash
# Через Homebrew (рекомендуется)
brew install gh

# Через MacPorts
sudo port install gh

# Через Conda
conda install gh --channel conda-forge
```

### Linux

```bash
# Ubuntu/Debian
type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y)
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
&& sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
&& sudo apt update \
&& sudo apt install gh -y

# Fedora/RHEL/CentOS
sudo dnf install gh

# Arch Linux
sudo pacman -S github-cli

# openSUSE
sudo zypper install gh
```

### Windows

```powershell
# Через winget (рекомендуется)
winget install --id GitHub.cli

# Через Chocolatey
choco install gh

# Через Scoop
scoop install gh
```

### Проверка установки

```bash
gh --version
# gh version 2.x.x (20xx-xx-xx)
```

---

## Авторизация

Перед использованием gh необходимо авторизоваться в GitHub:

```bash
gh auth login
```

### Интерактивный процесс авторизации:

```
? What account do you want to log into?
> GitHub.com
  GitHub Enterprise Server

? What is your preferred protocol for Git operations?
> HTTPS
  SSH

? Authenticate Git with your GitHub credentials? (Y/n) Y

? How would you like to authenticate GitHub CLI?
> Login with a web browser
  Paste an authentication token
```

### Авторизация через браузер (рекомендуется):
1. gh откроет браузер
2. Введите одноразовый код, показанный в терминале
3. Подтвердите авторизацию в браузере

### Авторизация через токен:
```bash
# Создайте Personal Access Token на GitHub
# Settings → Developer settings → Personal access tokens

gh auth login --with-token < mytoken.txt
# или
echo "ghp_xxxxxxxxxxxx" | gh auth login --with-token
```

### Проверка статуса авторизации:
```bash
gh auth status
```

### Выход из аккаунта:
```bash
gh auth logout
```

### Обновление токена:
```bash
gh auth refresh
```

---

## Основные команды

### gh repo — работа с репозиториями

#### Создание репозитория

```bash
# Создать публичный репозиторий
gh repo create my-project --public

# Создать приватный репозиторий
gh repo create my-project --private

# Создать из текущей директории
gh repo create --source=. --public

# Интерактивное создание
gh repo create

# Создать с описанием и README
gh repo create my-project --public --description "My awesome project" --add-readme
```

#### Клонирование репозитория

```bash
# Клонировать свой репозиторий
gh repo clone my-project

# Клонировать чужой репозиторий
gh repo clone owner/repo

# Клонировать в определенную директорию
gh repo clone owner/repo ./my-directory
```

#### Форк репозитория

```bash
# Создать форк
gh repo fork owner/repo

# Создать форк и клонировать
gh repo fork owner/repo --clone

# Форк без клонирования
gh repo fork owner/repo --clone=false
```

#### Просмотр информации о репозитории

```bash
# Открыть в браузере
gh repo view --web

# Показать информацию в терминале
gh repo view

# Посмотреть конкретный репозиторий
gh repo view owner/repo

# Показать только README
gh repo view --branch main
```

#### Другие операции

```bash
# Список ваших репозиториев
gh repo list

# Список репозиториев организации
gh repo list my-org

# Удалить репозиторий
gh repo delete my-project --yes

# Переименовать репозиторий
gh repo rename new-name

# Архивировать репозиторий
gh repo archive my-project
```

---

### gh issue — работа с issues

#### Создание issue

```bash
# Интерактивное создание
gh issue create

# Создать с заголовком и телом
gh issue create --title "Bug: Login не работает" --body "Описание бага..."

# Создать с labels и assignee
gh issue create --title "Новая фича" --label "enhancement" --assignee @me

# Создать из файла
gh issue create --title "Bug report" --body-file bug-description.md
```

#### Просмотр issues

```bash
# Список открытых issues
gh issue list

# Список всех issues (включая закрытые)
gh issue list --state all

# Только закрытые issues
gh issue list --state closed

# Фильтр по label
gh issue list --label "bug"

# Фильтр по assignee
gh issue list --assignee @me

# Просмотр конкретного issue
gh issue view 123

# Открыть issue в браузере
gh issue view 123 --web
```

#### Управление issues

```bash
# Закрыть issue
gh issue close 123

# Закрыть с комментарием
gh issue close 123 --comment "Исправлено в PR #456"

# Переоткрыть issue
gh issue reopen 123

# Редактировать issue
gh issue edit 123 --title "Новый заголовок"

# Добавить labels
gh issue edit 123 --add-label "priority:high"

# Назначить на себя
gh issue edit 123 --add-assignee @me

# Добавить комментарий
gh issue comment 123 --body "Работаю над этим!"

# Перенести в другой репозиторий
gh issue transfer 123 owner/other-repo

# Закрепить issue
gh issue pin 123
```

#### Поиск issues

```bash
# Поиск по ключевым словам
gh issue list --search "login error"

# Поиск с фильтрами
gh issue list --search "is:open label:bug author:username"
```

---

### gh pr — работа с Pull Requests

#### Создание Pull Request

```bash
# Интерактивное создание
gh pr create

# Создать с заголовком и описанием
gh pr create --title "Добавил новую фичу" --body "Описание изменений..."

# Создать как черновик
gh pr create --draft

# Указать base и head ветки
gh pr create --base main --head feature-branch

# Создать с reviewers
gh pr create --reviewer user1,user2

# Создать с labels
gh pr create --label "enhancement" --label "needs-review"

# Заполнить из шаблона
gh pr create --fill
```

#### Просмотр Pull Requests

```bash
# Список открытых PR
gh pr list

# Все PR (включая merged и closed)
gh pr list --state all

# Только ваши PR
gh pr list --author @me

# PR для review
gh pr list --search "review-requested:@me"

# Просмотр конкретного PR
gh pr view 456

# Просмотр diff
gh pr diff 456

# Открыть в браузере
gh pr view 456 --web
```

#### Checkout Pull Request

```bash
# Переключиться на ветку PR
gh pr checkout 456

# Checkout по URL
gh pr checkout https://github.com/owner/repo/pull/456

# Checkout в новую локальную ветку
gh pr checkout 456 --branch my-local-branch
```

#### Code Review

```bash
# Approve PR
gh pr review 456 --approve

# Request changes
gh pr review 456 --request-changes --body "Нужно исправить..."

# Оставить комментарий
gh pr review 456 --comment --body "Выглядит хорошо, но есть вопросы"

# Интерактивный review
gh pr review 456
```

#### Merge Pull Request

```bash
# Merge (по умолчанию создает merge commit)
gh pr merge 456

# Squash and merge
gh pr merge 456 --squash

# Rebase and merge
gh pr merge 456 --rebase

# Merge и удалить ветку
gh pr merge 456 --delete-branch

# Auto-merge после прохождения checks
gh pr merge 456 --auto --squash
```

#### Управление PR

```bash
# Закрыть без merge
gh pr close 456

# Переоткрыть PR
gh pr reopen 456

# Редактировать PR
gh pr edit 456 --title "Новый заголовок"

# Добавить reviewers
gh pr edit 456 --add-reviewer user1

# Конвертировать draft в ready
gh pr ready 456

# Проверить статус checks
gh pr checks 456

# Ждать пока checks пройдут
gh pr checks 456 --watch
```

---

### gh gist — работа с Gists

Gists — это способ быстро поделиться кодом или заметками.

```bash
# Создать публичный gist из файла
gh gist create script.py

# Создать приватный gist
gh gist create script.py --public=false

# Создать с описанием
gh gist create script.py --desc "Полезный скрипт"

# Создать из нескольких файлов
gh gist create file1.py file2.js

# Создать из stdin
echo "Hello World" | gh gist create -

# Список ваших gists
gh gist list

# Просмотреть gist
gh gist view GIST_ID

# Редактировать gist
gh gist edit GIST_ID

# Клонировать gist
gh gist clone GIST_ID

# Удалить gist
gh gist delete GIST_ID
```

---

### gh workflow — работа с GitHub Actions

```bash
# Список workflows
gh workflow list

# Просмотр workflow
gh workflow view "CI"

# Запустить workflow вручную
gh workflow run "CI"

# Запустить с параметрами
gh workflow run "Deploy" -f environment=production

# Запустить на определенной ветке
gh workflow run "CI" --ref feature-branch

# Отключить workflow
gh workflow disable "CI"

# Включить workflow
gh workflow enable "CI"
```

#### Просмотр запусков (runs)

```bash
# Список последних запусков
gh run list

# Фильтр по workflow
gh run list --workflow "CI"

# Фильтр по статусу
gh run list --status failure

# Просмотр конкретного запуска
gh run view 123456789

# Смотреть логи в реальном времени
gh run watch 123456789

# Скачать артефакты
gh run download 123456789

# Перезапустить failed jobs
gh run rerun 123456789 --failed

# Отменить запуск
gh run cancel 123456789
```

---

### gh release — работа с релизами

```bash
# Создать релиз
gh release create v1.0.0

# Создать с заголовком и описанием
gh release create v1.0.0 --title "Version 1.0.0" --notes "Первый релиз!"

# Создать из файла с release notes
gh release create v1.0.0 --notes-file CHANGELOG.md

# Автоматически сгенерировать release notes
gh release create v1.0.0 --generate-notes

# Создать pre-release
gh release create v1.0.0-beta --prerelease

# Создать draft релиз
gh release create v1.0.0 --draft

# Прикрепить файлы (assets)
gh release create v1.0.0 ./dist/app.zip ./dist/app.tar.gz

# Список релизов
gh release list

# Просмотр релиза
gh release view v1.0.0

# Скачать assets из релиза
gh release download v1.0.0

# Удалить релиз
gh release delete v1.0.0 --yes

# Редактировать релиз
gh release edit v1.0.0 --draft=false
```

---

## Практические примеры

### Сценарий 1: Полный цикл работы над фичей

```bash
# 1. Клонируем репозиторий
gh repo clone my-org/project
cd project

# 2. Создаем ветку и вносим изменения
git checkout -b feature/user-auth
# ... пишем код ...
git add .
git commit -m "feat: add user authentication"

# 3. Пушим и создаем PR
git push -u origin feature/user-auth
gh pr create --title "feat: User Authentication" \
  --body "Добавлена авторизация пользователей" \
  --reviewer team-lead \
  --label "feature"

# 4. Следим за проверками
gh pr checks --watch

# 5. После approve мержим
gh pr merge --squash --delete-branch
```

### Сценарий 2: Быстрый фикс бага

```bash
# 1. Находим issue
gh issue list --label "bug" --state open

# 2. Берем issue на себя
gh issue edit 42 --add-assignee @me

# 3. Создаем ветку, фиксим, пушим
git checkout -b fix/issue-42
# ... фиксим баг ...
git commit -am "fix: resolve issue #42"
git push -u origin fix/issue-42

# 4. Создаем PR, который закроет issue
gh pr create --title "fix: Login button not working" \
  --body "Closes #42"

# 5. Мержим после review
gh pr merge --squash --delete-branch
```

### Сценарий 3: Code Review

```bash
# 1. Смотрим PR на review
gh pr list --search "review-requested:@me"

# 2. Checkout PR локально для тестирования
gh pr checkout 123

# 3. Смотрим изменения
gh pr diff 123

# 4. Запускаем тесты локально
npm test

# 5. Approve или request changes
gh pr review 123 --approve --body "LGTM! Отличная работа"
# или
gh pr review 123 --request-changes --body "Нужно добавить тесты для edge cases"
```

### Сценарий 4: Работа с форками (Open Source)

```bash
# 1. Форкаем и клонируем популярный проект
gh repo fork facebook/react --clone
cd react

# 2. Создаем ветку для изменений
git checkout -b fix/typo-in-docs

# 3. Вносим изменения
# ... редактируем файлы ...
git commit -am "docs: fix typo in README"
git push -u origin fix/typo-in-docs

# 4. Создаем PR в оригинальный репозиторий
gh pr create --repo facebook/react \
  --title "docs: fix typo in README" \
  --body "Fixed small typo in documentation"
```

### Сценарий 5: Релиз новой версии

```bash
# 1. Проверяем что все PR замержены
gh pr list --state open

# 2. Создаем тег
git tag v2.0.0
git push origin v2.0.0

# 3. Создаем релиз с автоматическими release notes
gh release create v2.0.0 \
  --title "Version 2.0.0" \
  --generate-notes \
  ./dist/app-linux.tar.gz \
  ./dist/app-macos.tar.gz \
  ./dist/app-windows.zip

# 4. Проверяем что релиз создан
gh release view v2.0.0 --web
```

### Сценарий 6: Мониторинг CI/CD

```bash
# 1. Смотрим последние запуски
gh run list --limit 10

# 2. Проверяем статус текущего PR
gh pr checks

# 3. Если что-то упало, смотрим логи
gh run view 123456789 --log-failed

# 4. Перезапускаем failed jobs
gh run rerun 123456789 --failed

# 5. Следим за новым запуском
gh run watch
```

---

## Алиасы и настройка

### Создание алиасов

Алиасы позволяют сокращать часто используемые команды:

```bash
# Создать алиас
gh alias set prc 'pr create'

# Теперь можно использовать
gh prc --title "My PR"

# Алиас с параметрами
gh alias set co 'pr checkout'
gh co 123

# Сложный алиас
gh alias set bugs 'issue list --label "bug" --state open'
gh bugs

# Алиас для просмотра своих PR
gh alias set mypr 'pr list --author @me'

# Просмотр всех алиасов
gh alias list

# Удалить алиас
gh alias delete prc
```

### Полезные алиасы

```bash
# Быстрое создание PR
gh alias set prd 'pr create --draft'

# Список PR на review
gh alias set review 'pr list --search "review-requested:@me"'

# Мои открытые issues
gh alias set myissues 'issue list --assignee @me --state open'

# Быстрый merge
gh alias set pm 'pr merge --squash --delete-branch'

# Статус CI для текущего PR
gh alias set ci 'pr checks'

# Смотреть логи последнего run
gh alias set logs 'run view --log'
```

### Настройка конфигурации

```bash
# Установить редактор по умолчанию
gh config set editor "code --wait"

# Установить браузер
gh config set browser "firefox"

# Предпочитаемый протокол для Git
gh config set git_protocol ssh

# Отключить подсказки
gh config set prompt disabled

# Посмотреть текущую конфигурацию
gh config list

# Конфигурация хранится в ~/.config/gh/config.yml
```

### Настройка формата вывода

```bash
# JSON вывод для скриптов
gh issue list --json number,title,state

# Форматирование с jq
gh issue list --json number,title | jq '.[] | "\(.number): \(.title)"'

# Template formatting
gh pr list --json number,title,author --template \
  '{{range .}}#{{.number}} {{.title}} by {{.author.login}}{{"\n"}}{{end}}'
```

---

## Преимущества перед веб-интерфейсом

### 1. Скорость работы
- Не нужно переключаться между терминалом и браузером
- Быстрое выполнение рутинных операций
- Автодополнение команд (Tab)

### 2. Автоматизация
```bash
# Скрипт для создания релиза
#!/bin/bash
VERSION=$1
gh release create $VERSION \
  --generate-notes \
  --title "Release $VERSION" \
  ./dist/*
```

### 3. Интеграция с Git
```bash
# Сразу после коммита создать PR
git commit -am "fix: bug"
git push
gh pr create --fill
```

### 4. Работа с несколькими аккаунтами
```bash
# Переключение между аккаунтами
gh auth switch

# Использование разных хостов
gh auth login --hostname github.mycompany.com
```

### 5. Offline-first workflow
- Можно подготовить команды заранее
- Batch операции через скрипты
- История команд в терминале

### 6. CI/CD интеграция
```yaml
# В GitHub Actions
- name: Create Release
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: gh release create ${{ github.ref_name }} --generate-notes
```

### Сравнительная таблица

| Операция | Веб-интерфейс | GitHub CLI |
|----------|---------------|------------|
| Создать PR | ~1 минута (переход, заполнение форм) | ~5 секунд |
| Merge PR | Несколько кликов | Одна команда |
| Посмотреть CI статус | Открыть браузер, найти PR | `gh pr checks` |
| Создать issue из шаблона | Выбор шаблона, заполнение | `gh issue create` |
| Массовые операции | Невозможно | Легко через скрипты |

---

## Дополнительные возможности

### gh api — прямой доступ к GitHub API

```bash
# GET запрос
gh api repos/owner/repo

# POST запрос
gh api repos/owner/repo/issues -f title="Bug" -f body="Description"

# С пагинацией
gh api repos/owner/repo/issues --paginate

# GraphQL запросы
gh api graphql -f query='
  query {
    viewer {
      login
      repositories(first: 10) {
        nodes { name }
      }
    }
  }
'
```

### gh extension — расширения

```bash
# Поиск расширений
gh extension search

# Установка расширения
gh extension install dlvhdr/gh-dash

# Список установленных
gh extension list

# Обновление расширений
gh extension upgrade --all
```

### Популярные расширения:
- `gh-dash` — интерактивный dashboard для PR и issues
- `gh-copilot` — интеграция с GitHub Copilot
- `gh-poi` — очистка локальных merged веток
- `gh-markdown-preview` — превью markdown файлов

---

## Полезные советы

1. **Используйте `--help`** для любой команды:
   ```bash
   gh pr --help
   gh pr create --help
   ```

2. **Tab-completion** — настройте автодополнение:
   ```bash
   # Для bash
   gh completion -s bash > /etc/bash_completion.d/gh

   # Для zsh
   gh completion -s zsh > ~/.zsh/completions/_gh
   ```

3. **Переменная GH_TOKEN** — для скриптов:
   ```bash
   export GH_TOKEN=ghp_xxxx
   gh pr list
   ```

4. **Работа с организациями**:
   ```bash
   gh repo list my-org --limit 100
   gh issue list --repo my-org/project
   ```

5. **JSON output для парсинга**:
   ```bash
   gh pr list --json number,title,state --jq '.[] | select(.state=="OPEN")'
   ```
