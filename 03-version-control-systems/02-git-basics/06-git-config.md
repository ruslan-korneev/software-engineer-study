# Git Config (Конфигурация Git)

## Введение

**Git config** — система настроек Git, которая позволяет персонализировать поведение Git под ваши нужды. Настройки хранятся на трёх уровнях с разным приоритетом.

## Уровни конфигурации

### 1. System (Системный)
Применяется ко всем пользователям системы.

```bash
# Расположение: /etc/gitconfig (Linux/macOS) или C:\ProgramData\Git\config (Windows)

# Просмотр
git config --system --list

# Изменение (требует sudo/admin)
sudo git config --system core.editor vim
```

### 2. Global (Глобальный)
Применяется ко всем репозиториям текущего пользователя.

```bash
# Расположение: ~/.gitconfig или ~/.config/git/config

# Просмотр
git config --global --list

# Изменение
git config --global user.name "Ivan Petrov"
```

### 3. Local (Локальный)
Применяется только к текущему репозиторию. **Имеет наивысший приоритет.**

```bash
# Расположение: .git/config в репозитории

# Просмотр
git config --local --list

# Изменение
git config --local user.email "ivan@company.com"
```

### Приоритет настроек

```
Local > Global > System
```

Если одна настройка задана на нескольких уровнях, используется значение с более высоким приоритетом (local).

## Основные команды

### Просмотр настроек

```bash
# Все настройки со всех уровней
git config --list

# Показать источник каждой настройки
git config --list --show-origin

# Показать уровень каждой настройки
git config --list --show-scope

# Конкретная настройка
git config user.name
git config user.email

# Конкретная настройка с уровнем
git config --show-origin user.name
```

### Установка настроек

```bash
# Глобальные настройки
git config --global user.name "Ivan Petrov"
git config --global user.email "ivan@example.com"

# Локальные настройки (для конкретного репозитория)
git config --local user.email "ivan@company.com"

# Без флага — по умолчанию --local
git config user.email "ivan@company.com"
```

### Удаление настроек

```bash
# Удалить настройку
git config --global --unset user.name

# Удалить секцию целиком
git config --global --remove-section alias
```

### Редактирование файла напрямую

```bash
# Открыть глобальный конфиг в редакторе
git config --global --edit

# Открыть локальный конфиг
git config --local --edit
```

## Обязательные настройки

### Идентификация пользователя

```bash
# Имя (отображается в коммитах)
git config --global user.name "Ivan Petrov"

# Email (связывает коммиты с аккаунтом GitHub/GitLab)
git config --global user.email "ivan@example.com"

# Для рабочих проектов — другой email
cd work-project/
git config --local user.email "ivan@company.com"
```

## Рекомендуемые настройки

### Редактор по умолчанию

```bash
# VS Code
git config --global core.editor "code --wait"

# Vim
git config --global core.editor "vim"

# Nano
git config --global core.editor "nano"

# Sublime Text
git config --global core.editor "subl -n -w"

# Notepad++ (Windows)
git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
```

### Ветка по умолчанию

```bash
# Использовать "main" вместо "master" для новых репозиториев
git config --global init.defaultBranch main
```

### Поведение push

```bash
# Пушить только текущую ветку (рекомендуется)
git config --global push.default current

# Или более строгий вариант — пушить только если upstream совпадает
git config --global push.default simple
```

### Поведение pull

```bash
# Использовать rebase вместо merge при pull
git config --global pull.rebase true

# Или только fast-forward (самый безопасный)
git config --global pull.ff only
```

### Окончания строк

```bash
# Для Windows (конвертировать LF в CRLF при checkout)
git config --global core.autocrlf true

# Для macOS/Linux (оставлять LF)
git config --global core.autocrlf input

# Отключить конвертацию (не рекомендуется для кросс-платформенных проектов)
git config --global core.autocrlf false
```

### Цвета в терминале

```bash
# Включить цветной вывод
git config --global color.ui auto

# Настроить цвета для diff
git config --global color.diff.meta "blue"
git config --global color.diff.old "red"
git config --global color.diff.new "green"
```

### Diff и Merge инструменты

```bash
# Использовать VS Code для diff
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Использовать VS Code для merge
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

## Алиасы (Aliases)

Алиасы — сокращения для часто используемых команд.

### Основные алиасы

```bash
# Короткие версии команд
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status

# Использование
git co feature    # вместо git checkout feature
git st            # вместо git status
```

### Продвинутые алиасы

```bash
# Красивый лог
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

# Последний коммит
git config --global alias.last "log -1 HEAD --stat"

# Отмена последнего коммита (сохраняя изменения)
git config --global alias.undo "reset HEAD~1 --mixed"

# Показать все алиасы
git config --global alias.aliases "config --get-regexp ^alias\."

# Поиск в истории
git config --global alias.find "log --all --oneline --grep"

# Добавить всё и закоммитить
git config --global alias.ac "!git add -A && git commit -m"
# Использование: git ac "commit message"

# Текущая ветка
git config --global alias.current "rev-parse --abbrev-ref HEAD"

# Список веток с датой последнего коммита
git config --global alias.recent "branch --sort=-committerdate --format='%(committerdate:relative)%09%(refname:short)'"
```

### Алиасы с внешними командами

```bash
# ! в начале позволяет выполнять shell-команды
git config --global alias.visual "!gitk"

# Открыть репозиторий в браузере (GitHub)
git config --global alias.web "!open $(git remote get-url origin | sed 's/git@github.com:/https:\\/\\/github.com\\//' | sed 's/.git$//')"
```

## Пример полного ~/.gitconfig

```ini
[user]
    name = Ivan Petrov
    email = ivan@example.com

[init]
    defaultBranch = main

[core]
    editor = code --wait
    autocrlf = input
    excludesfile = ~/.gitignore_global

[push]
    default = current
    autoSetupRemote = true

[pull]
    rebase = true

[fetch]
    prune = true

[diff]
    colorMoved = zebra

[merge]
    conflictstyle = diff3

[color]
    ui = auto

[alias]
    # Основные
    co = checkout
    br = branch
    ci = commit
    st = status

    # Лог
    lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
    last = log -1 HEAD --stat

    # Утилиты
    undo = reset HEAD~1 --mixed
    amend = commit --amend --no-edit
    aliases = config --get-regexp ^alias\\.

    # Ветки
    branches = branch -a
    current = rev-parse --abbrev-ref HEAD
    recent = branch --sort=-committerdate --format='%(committerdate:relative)%09%(refname:short)'

[credential]
    helper = osxkeychain  # для macOS
    # helper = manager    # для Windows

[filter "lfs"]
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true
```

## Условные конфигурации

Git 2.13+ поддерживает условные настройки на основе пути.

```ini
# ~/.gitconfig

[user]
    name = Ivan Petrov
    email = ivan@personal.com

# Для рабочих проектов
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

# Для опенсорс проектов
[includeIf "gitdir:~/opensource/"]
    path = ~/.gitconfig-opensource
```

```ini
# ~/.gitconfig-work
[user]
    email = ivan@company.com
    signingkey = WORK_KEY_ID

[commit]
    gpgsign = true
```

## Полезные настройки

### Автоматическое исправление опечаток

```bash
# Автоматически исправлять команды через 1.5 секунды
git config --global help.autocorrect 15

# git stats → Did you mean 'status'?
# После 1.5 секунд выполнит git status
```

### Повторное использование записанных решений (rerere)

```bash
# Запоминать, как вы разрешали конфликты
git config --global rerere.enabled true

# При повторении такого же конфликта Git применит решение автоматически
```

### Подпись коммитов GPG

```bash
# Настроить ключ GPG
git config --global user.signingkey YOUR_GPG_KEY_ID

# Подписывать все коммиты
git config --global commit.gpgsign true

# Подписывать все теги
git config --global tag.gpgsign true
```

### Кэширование учётных данных

```bash
# macOS (использовать Keychain)
git config --global credential.helper osxkeychain

# Linux (кэшировать в памяти на 1 час)
git config --global credential.helper 'cache --timeout=3600'

# Windows (использовать Credential Manager)
git config --global credential.helper manager
```

## Best Practices

### 1. Настройте идентификацию сразу
```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 2. Используйте алиасы
Экономьте время на частых командах.

### 3. Настройте редактор
```bash
git config --global core.editor "code --wait"
```

### 4. Используйте условные конфигурации
Разные email для работы и личных проектов.

### 5. Включите rerere
```bash
git config --global rerere.enabled true
```

### 6. Создайте глобальный .gitignore
```bash
git config --global core.excludesfile ~/.gitignore_global
```

## Типичные ошибки

### 1. Забыли настроить user.name/email
```bash
# Git откажется делать коммиты
# "Please tell me who you are"

git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 2. Неправильный email для GitHub
```bash
# Коммиты не связываются с аккаунтом GitHub
# Убедитесь, что email совпадает с email в настройках GitHub
git config user.email  # проверить текущий
```

### 3. Конфликт локальных и глобальных настроек
```bash
# Проверить, откуда берётся значение
git config --show-origin user.email
```

### 4. Забыли --wait для редактора
```bash
# ПЛОХО — Git не дождётся закрытия редактора
git config --global core.editor "code"

# ХОРОШО
git config --global core.editor "code --wait"
```

## Диагностика проблем

```bash
# Показать все настройки с источниками
git config --list --show-origin --show-scope

# Проверить конкретную настройку
git config --show-origin user.email

# Проверить, применяется ли алиас
git config --get alias.lg

# Валидация конфигурации
git config --list
```

## Резюме

- **Три уровня:** system < global < local (по приоритету)
- **Обязательно настройте:** `user.name` и `user.email`
- **Рекомендуется:** `core.editor`, `init.defaultBranch`, `push.default`
- **Алиасы** — сокращения для частых команд (`git st` вместо `git status`)
- **Условные конфигурации** — разные настройки для разных проектов
- `git config --list --show-origin` — отладка настроек
- Храните полезные настройки в `~/.gitconfig` и синхронизируйте между машинами
