# Установка Git

## Проверка установки

Перед установкой проверьте, не установлен ли Git уже:

```bash
git --version
# Если установлен: git version 2.43.0
# Если нет: command not found: git
```

## Установка на разных ОС

### macOS

#### Способ 1: Xcode Command Line Tools (рекомендуется)
```bash
# При первом вызове git предложит установку
git --version

# Или установить явно
xcode-select --install
```

#### Способ 2: Homebrew
```bash
# Установить Homebrew (если нет)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Установить Git
brew install git
```

#### Способ 3: Официальный установщик
Скачать с https://git-scm.com/download/mac

### Linux

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install git
```

#### Fedora
```bash
sudo dnf install git
```

#### CentOS/RHEL
```bash
sudo yum install git
# или для новых версий
sudo dnf install git
```

#### Arch Linux
```bash
sudo pacman -S git
```

#### OpenSUSE
```bash
sudo zypper install git
```

### Windows

#### Способ 1: Git for Windows (рекомендуется)
1. Скачать установщик с https://git-scm.com/download/win
2. Запустить установщик
3. Следовать инструкциям (рекомендуемые настройки по умолчанию)

**Важные опции при установке:**
- **Редактор:** VS Code (или ваш предпочтительный)
- **PATH:** "Git from the command line and also from 3rd-party software"
- **SSH:** "Use bundled OpenSSH"
- **HTTPS:** "Use the OpenSSL library"
- **Line endings:** "Checkout Windows-style, commit Unix-style"
- **Terminal:** "Use MinTTY" (рекомендуется)

#### Способ 2: Windows Package Manager (winget)
```powershell
winget install --id Git.Git -e --source winget
```

#### Способ 3: Chocolatey
```powershell
choco install git
```

#### Способ 4: WSL (Windows Subsystem for Linux)
```bash
# В WSL Ubuntu
sudo apt update
sudo apt install git
```

## Первоначальная настройка

После установки **обязательно** настройте Git:

### Имя и email (ОБЯЗАТЕЛЬНО)

```bash
# Ваше имя (будет в коммитах)
git config --global user.name "Ваше Имя"

# Ваш email (должен совпадать с email на GitHub/GitLab)
git config --global user.email "your.email@example.com"
```

**Важно:** Эти данные записываются в каждый коммит и видны всем.

### Проверка настроек

```bash
# Показать все настройки
git config --list

# Показать конкретную настройку
git config user.name
git config user.email
```

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

### Настройка переносов строк

```bash
# Windows: конвертировать LF в CRLF при checkout
git config --global core.autocrlf true

# macOS/Linux: конвертировать CRLF в LF при commit
git config --global core.autocrlf input

# Отключить (не рекомендуется в смешанных командах)
git config --global core.autocrlf false
```

### Настройка ветки по умолчанию

```bash
# Использовать 'main' вместо 'master' для новых репозиториев
git config --global init.defaultBranch main
```

### Цветной вывод

```bash
git config --global color.ui auto
```

### Полезные алиасы

```bash
# Короткие команды
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"

# Использование
git st      # вместо git status
git co main # вместо git checkout main
git lg      # красивый лог с графом
```

## Конфигурационные файлы

Git хранит настройки в трёх местах:

### Уровни конфигурации

| Уровень | Файл | Область |
|---------|------|---------|
| System | `/etc/gitconfig` | Все пользователи системы |
| Global | `~/.gitconfig` | Текущий пользователь |
| Local | `.git/config` | Текущий репозиторий |

**Приоритет:** Local > Global > System

### Просмотр конфигурации

```bash
# Показать все настройки с указанием источника
git config --list --show-origin

# Редактировать global конфиг
git config --global --edit

# Редактировать local конфиг (в репозитории)
git config --local --edit
```

### Пример ~/.gitconfig

```ini
[user]
    name = Иван Петров
    email = ivan@example.com

[core]
    editor = code --wait
    autocrlf = input

[init]
    defaultBranch = main

[color]
    ui = auto

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --all
    last = log -1 HEAD
    unstage = reset HEAD --

[pull]
    rebase = false

[push]
    default = current
```

## Настройка SSH для GitHub/GitLab

### Генерация SSH ключа

```bash
# Генерация ключа (используйте ваш email)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Для старых систем без поддержки Ed25519
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"
```

**Процесс:**
```
Enter file in which to save the key (/home/user/.ssh/id_ed25519): [Enter]
Enter passphrase (empty for no passphrase): [введите пароль]
Enter same passphrase again: [повторите пароль]
```

### Добавление ключа в ssh-agent

```bash
# Запуск ssh-agent
eval "$(ssh-agent -s)"

# Добавление ключа
ssh-add ~/.ssh/id_ed25519
```

### Копирование публичного ключа

```bash
# macOS
cat ~/.ssh/id_ed25519.pub | pbcopy

# Linux
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard

# Windows (Git Bash)
cat ~/.ssh/id_ed25519.pub | clip

# Или просто вывести и скопировать вручную
cat ~/.ssh/id_ed25519.pub
```

### Добавление ключа на GitHub

1. Перейти на GitHub → Settings → SSH and GPG keys
2. Нажать "New SSH key"
3. Title: "My Laptop" (или любое описание)
4. Key: вставить скопированный публичный ключ
5. Нажать "Add SSH key"

### Проверка подключения

```bash
# GitHub
ssh -T git@github.com
# Ответ: Hi username! You've successfully authenticated...

# GitLab
ssh -T git@gitlab.com

# Bitbucket
ssh -T git@bitbucket.org
```

## Обновление Git

### macOS (Homebrew)
```bash
brew upgrade git
```

### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt upgrade git
```

### Windows
Скачать новый установщик с https://git-scm.com или:
```bash
git update-git-for-windows
```

## Проверка установки

После установки и настройки проверьте:

```bash
# Версия
git --version

# Настройки
git config --list

# Пользователь
git config user.name
git config user.email

# SSH (если настроили)
ssh -T git@github.com
```

## Типичные проблемы

### Git не найден после установки (Windows)
**Причина:** Git не добавлен в PATH
**Решение:** Переустановить с опцией "Git from the command line..."

### Permission denied (publickey)
**Причина:** SSH ключ не настроен или не добавлен
**Решение:**
```bash
# Проверить наличие ключа
ls -la ~/.ssh

# Если нет - сгенерировать
ssh-keygen -t ed25519 -C "email@example.com"

# Добавить на GitHub/GitLab
```

### SSL certificate problem
**Причина:** Проблемы с сертификатами (часто в корпоративных сетях)
**Временное решение (не рекомендуется для постоянного использования):**
```bash
git config --global http.sslVerify false
```

### Неправильная кодировка в именах файлов
```bash
git config --global core.quotePath false
```

## Best Practices

1. **Используйте SSH вместо HTTPS** — безопаснее и удобнее
2. **Настройте редактор** — избежите проблем с Vim
3. **Установите алиасы** — экономьте время
4. **Обновляйте Git** — новые версии быстрее и безопаснее
5. **Используйте passphrase для SSH** — дополнительная защита

## Резюме

Минимальная настройка после установки:

```bash
# 1. Проверить версию
git --version

# 2. Настроить имя и email
git config --global user.name "Ваше Имя"
git config --global user.email "your@email.com"

# 3. Настроить редактор
git config --global core.editor "code --wait"

# 4. Настроить ветку по умолчанию
git config --global init.defaultBranch main

# 5. (Опционально) Настроить SSH для GitHub
ssh-keygen -t ed25519 -C "your@email.com"
# Добавить публичный ключ на GitHub
```

Теперь вы готовы работать с Git!
