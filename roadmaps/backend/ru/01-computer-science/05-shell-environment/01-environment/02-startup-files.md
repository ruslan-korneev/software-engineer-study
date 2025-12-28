# Файлы запуска оболочки (Startup Files)

[prev: 01-environment-variables](./01-environment-variables.md) | [next: 03-modifying-environment](./03-modifying-environment.md)

---
## Типы сессий оболочки

Прежде чем разбираться с файлами запуска, нужно понять типы сессий shell:

### Login Shell (оболочка входа)

**Login shell** запускается при:
- Входе в систему через консоль (Ctrl+Alt+F1)
- Подключении через SSH
- Использовании `su - username`
- Использовании `bash --login`

Признаки login shell:
```bash
# Проверить, является ли shell login-оболочкой
echo $0
# -bash (минус в начале означает login shell)

# Или использовать shopt
shopt -q login_shell && echo "Login shell" || echo "Non-login shell"
```

### Non-login Shell (интерактивная оболочка)

**Non-login shell** запускается при:
- Открытии нового окна терминала в GUI
- Запуске нового shell командой `bash`
- Открытии новой вкладки в терминале

### Интерактивная vs Неинтерактивная

**Интерактивная оболочка** — вы вводите команды вручную.
**Неинтерактивная оболочка** — выполняет скрипт без взаимодействия с пользователем.

```bash
# Проверить интерактивность
[[ $- == *i* ]] && echo "Interactive" || echo "Non-interactive"
```

## Файлы запуска Bash

### Порядок чтения файлов

#### Для Login Shell:
```
1. /etc/profile          (системный, для всех пользователей)
2. ~/.bash_profile       (пользовательский, первый найденный)
   ИЛИ ~/.bash_login     (если .bash_profile не существует)
   ИЛИ ~/.profile        (если предыдущие не существуют)
```

#### Для Non-login Interactive Shell:
```
1. /etc/bash.bashrc      (системный, на некоторых дистрибутивах)
2. ~/.bashrc             (пользовательский)
```

#### При завершении Login Shell:
```
~/.bash_logout           (выполняется при выходе)
```

### Схема выбора файлов

```
                    +----------------+
                    |  Запуск Bash   |
                    +-------+--------+
                            |
              +-------------+-------------+
              |                           |
        Login Shell?                Non-login Shell?
              |                           |
    +---------+---------+         +-------+-------+
    |                   |         |               |
/etc/profile      ~/.bash_profile  /etc/bash.bashrc  ~/.bashrc
    |             (или .profile)         |
    +--------->  Читает ~/.bashrc <------+
                 (рекомендуется)
```

## Детальное описание файлов

### /etc/profile — системный профиль

Читается всеми login shells. Обычно содержит:
- Системные переменные окружения
- Настройки PATH для всех пользователей
- Подключение файлов из `/etc/profile.d/`

```bash
# Типичное содержимое /etc/profile
# Системный PATH
export PATH="/usr/local/bin:/usr/bin:/bin"

# Загрузка дополнительных скриптов
if [ -d /etc/profile.d ]; then
    for i in /etc/profile.d/*.sh; do
        if [ -r "$i" ]; then
            . "$i"
        fi
    done
fi
```

### ~/.bash_profile — профиль пользователя

Главный файл настроек для login shell. Рекомендуется хранить здесь:
- Переменные окружения
- Настройки PATH
- Вызов `.bashrc`

```bash
# ~/.bash_profile

# Добавить свои директории в PATH
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# Переменные окружения
export EDITOR="vim"
export LANG="en_US.UTF-8"

# Загрузить .bashrc для интерактивных настроек
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

### ~/.bashrc — настройки интерактивной оболочки

Читается при каждом запуске non-login shell. Содержит:
- Алиасы
- Функции
- Настройки промпта (PS1)
- Настройки истории команд

```bash
# ~/.bashrc

# Проверить, что shell интерактивный
[[ $- != *i* ]] && return

# Алиасы
alias ll='ls -la'
alias la='ls -A'
alias l='ls -CF'
alias grep='grep --color=auto'

# Настройки истории
HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth

# Настройка промпта
PS1='\u@\h:\w\$ '

# Автодополнение (если установлено)
if [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
fi
```

### ~/.profile — альтернатива .bash_profile

Используется если `.bash_profile` не существует. Совместим с другими shell (sh, dash):

```bash
# ~/.profile

# Добавить локальные bin-директории
if [ -d "$HOME/bin" ]; then
    PATH="$HOME/bin:$PATH"
fi

# Загрузить .bashrc если запущен bash
if [ -n "$BASH_VERSION" ]; then
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi
```

### ~/.bash_logout — действия при выходе

Выполняется при завершении login shell:

```bash
# ~/.bash_logout

# Очистить экран при выходе
clear

# Записать время выхода
echo "Logout: $(date)" >> ~/.logout_log
```

## Файлы запуска Zsh

Для zsh (по умолчанию в macOS) файлы немного отличаются:

```
Login shell:
1. /etc/zprofile
2. ~/.zprofile
3. ~/.zshrc
4. ~/.zlogin

Non-login shell:
1. ~/.zshrc

При выходе:
1. ~/.zlogout
```

```bash
# ~/.zshrc (основной файл для zsh)

# Алиасы
alias ll='ls -la'

# Настройка промпта
PROMPT='%n@%m:%~%# '

# История
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
```

## Практические рекомендации

### Правильная организация файлов

1. **~/.bash_profile** — только переменные окружения и вызов `.bashrc`:
```bash
# ~/.bash_profile
export PATH="$HOME/bin:$PATH"
export EDITOR="vim"

# Загрузить интерактивные настройки
[ -f ~/.bashrc ] && source ~/.bashrc
```

2. **~/.bashrc** — всё для интерактивной работы:
```bash
# ~/.bashrc
[[ $- != *i* ]] && return

# Алиасы, функции, промпт, история
alias ll='ls -la'
PS1='\u@\h:\w\$ '
```

### Проверка синтаксиса перед сохранением

```bash
# Проверить файл на ошибки
bash -n ~/.bashrc
# Если ошибок нет — тишина

# Тест в новом shell без сохранения
bash --rcfile ~/.bashrc_test
```

### Применение изменений

```bash
# Перечитать файл в текущем shell
source ~/.bashrc
# или
. ~/.bashrc

# Для .bash_profile нужен новый login shell
# или
source ~/.bash_profile
```

## Типичные ошибки

### Ошибка 1: Настройки не применяются

Причина: редактировали `.bash_profile`, а терминал читает `.bashrc`.

```bash
# Решение: добавить в .bash_profile
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

### Ошибка 2: Бесконечный цикл

```bash
# НЕПРАВИЛЬНО (в .bashrc):
source ~/.bash_profile  # Если .bash_profile вызывает .bashrc — цикл!

# ПРАВИЛЬНО:
# .bash_profile вызывает .bashrc, не наоборот
```

### Ошибка 3: Сломанный shell после редактирования

```bash
# Если .bashrc содержит ошибку, можно запустить shell без него:
bash --norc
# Исправить файл и перезапустить терминал
```

## Резюме

- Login shell читает `/etc/profile` и `~/.bash_profile`
- Non-login shell читает `~/.bashrc`
- Храните переменные окружения в `~/.bash_profile`
- Храните алиасы и функции в `~/.bashrc`
- В `~/.bash_profile` добавьте `source ~/.bashrc`
- Используйте `source` для применения изменений без перезапуска

---

[prev: 01-environment-variables](./01-environment-variables.md) | [next: 03-modifying-environment](./03-modifying-environment.md)
