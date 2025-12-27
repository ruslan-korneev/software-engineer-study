# Сохранение настроек Prompt (Saving Prompt)

## Временные изменения vs Постоянные

### Временное изменение

Изменения в текущей сессии терминала пропадут после закрытия:

```bash
# Установить prompt
PS1='\u@\h:\w\$ '

# Работает до закрытия терминала
```

### Постоянное сохранение

Для сохранения настроек между сессиями нужно добавить их в файлы конфигурации.

## Куда сохранять настройки

### ~/.bashrc — рекомендуемое место

Для Bash лучше всего сохранять PS1 в `~/.bashrc`:

```bash
# Открыть файл
vim ~/.bashrc

# Добавить в конец:
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '

# Применить изменения
source ~/.bashrc
```

### ~/.bash_profile

Можно использовать, но обычно он просто вызывает `.bashrc`:

```bash
# ~/.bash_profile
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

### ~/.profile

Для совместимости с разными shell (sh, dash):

```bash
# ~/.profile
PS1='\u@\h:\w\$ '
```

## Структура настройки в .bashrc

### Базовый вариант

```bash
# ~/.bashrc

# Проверить, что shell интерактивный
[[ $- != *i* ]] && return

# Настройка prompt
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

### С переменными для цветов

```bash
# ~/.bashrc

# Определение цветов
RED='\[\e[31m\]'
GREEN='\[\e[32m\]'
YELLOW='\[\e[33m\]'
BLUE='\[\e[34m\]'
RESET='\[\e[0m\]'
BOLD='\[\e[1m\]'

# Построение prompt
PS1="${GREEN}\u@\h${RESET}:${BLUE}\w${RESET}\$ "
```

### С функциями

```bash
# ~/.bashrc

# Функция для получения ветки git
parse_git_branch() {
    git branch 2>/dev/null | sed -n 's/* \(.*\)/(\1)/p'
}

# Функция для индикатора кода возврата
prompt_status() {
    if [ $? -eq 0 ]; then
        echo -e '\[\e[32m\]✓\[\e[0m\]'
    else
        echo -e '\[\e[31m\]✗\[\e[0m\]'
    fi
}

# Prompt с динамическими элементами
PROMPT_COMMAND='PS1="$(prompt_status) \[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\] \[\e[33m\]$(parse_git_branch)\[\e[0m\]\$ "'
```

## Пример полной конфигурации

```bash
# ~/.bashrc

# Выйти, если не интерактивный shell
[[ $- != *i* ]] && return

# ==============================
# Цвета и стили
# ==============================
RED='\[\e[31m\]'
GREEN='\[\e[32m\]'
YELLOW='\[\e[33m\]'
BLUE='\[\e[34m\]'
PURPLE='\[\e[35m\]'
CYAN='\[\e[36m\]'
WHITE='\[\e[37m\]'
RESET='\[\e[0m\]'
BOLD='\[\e[1m\]'

# ==============================
# Функции для prompt
# ==============================

# Git branch
__git_branch() {
    git branch 2>/dev/null | sed -n 's/* \(.*\)/ (\1)/p'
}

# Виртуальное окружение Python
__virtualenv() {
    if [ -n "$VIRTUAL_ENV" ]; then
        echo "($(basename $VIRTUAL_ENV)) "
    fi
}

# ==============================
# Prompt
# ==============================

# Разный prompt для root и обычного пользователя
if [ $UID -eq 0 ]; then
    # Root — красный
    PS1="${RED}\u@\h${RESET}:${BLUE}\w${RESET}# "
else
    # Обычный пользователь — зелёный
    PS1='$(__virtualenv)'"${GREEN}\u@\h${RESET}:${BLUE}\w${YELLOW}"'$(__git_branch)'"${RESET}\$ "
fi

# ==============================
# Prompt Command
# ==============================

# Записывать историю после каждой команды
PROMPT_COMMAND='history -a'

# Обновлять заголовок терминала
PROMPT_COMMAND+='; echo -ne "\033]0;${USER}@${HOSTNAME}: ${PWD}\007"'
```

## Применение изменений

### Способ 1: source

```bash
source ~/.bashrc
# или
. ~/.bashrc
```

### Способ 2: Перезапуск терминала

Закрыть и открыть новое окно терминала.

### Способ 3: exec

```bash
exec bash
# Заменяет текущий shell на новый
```

## Резервное копирование

Перед изменениями сделайте бэкап:

```bash
# Создать копию
cp ~/.bashrc ~/.bashrc.backup

# Если что-то сломалось — восстановить
cp ~/.bashrc.backup ~/.bashrc
source ~/.bashrc
```

## Отладка проблем

### Проблема: prompt не отображается правильно

```bash
# Сбросить к простому prompt
PS1='$ '

# Проверить escape-последовательности
# Убедиться, что \[ и \] правильно расставлены
```

### Проблема: строка переносится неправильно

Причина: escape-коды не обёрнуты в `\[` и `\]`.

```bash
# НЕПРАВИЛЬНО:
PS1='\e[32m\u\e[0m$ '

# ПРАВИЛЬНО:
PS1='\[\e[32m\]\u\[\e[0m\]$ '
```

### Проблема: изменения не сохраняются

Проверить:
1. Файл сохранён
2. Правильный файл редактируется (`.bashrc`, не `.bash_profile`)
3. Выполнен `source ~/.bashrc`

### Проблема: ошибки при запуске терминала

```bash
# Запустить bash без .bashrc
bash --norc

# Исправить ошибки в файле
vim ~/.bashrc

# Проверить синтаксис
bash -n ~/.bashrc
```

## Настройка для Zsh

В Zsh файл конфигурации — `~/.zshrc`:

```bash
# ~/.zshrc

# Zsh использует другой синтаксис
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f%# '

# Или PS1 (тоже работает)
PS1='%F{green}%n@%m%f:%F{blue}%~%f%# '

# С git branch
autoload -Uz vcs_info
precmd() { vcs_info }
zstyle ':vcs_info:git:*' formats ' (%b)'
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f%F{yellow}${vcs_info_msg_0_}%f%# '
```

## Готовые решения

### Starship (кроссплатформенный)

```bash
# Установка
curl -sS https://starship.rs/install.sh | sh

# Добавить в ~/.bashrc
eval "$(starship init bash)"
```

### Oh My Bash

```bash
# Установка
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ohmybash/oh-my-bash/master/tools/install.sh)"
```

### Powerlevel10k (для Zsh)

```bash
# Установка для Oh My Zsh
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

# В ~/.zshrc
ZSH_THEME="powerlevel10k/powerlevel10k"
```

## Чек-лист настройки

1. [ ] Создать бэкап `.bashrc`
2. [ ] Определить цвета в переменных
3. [ ] Написать prompt с `\[` и `\]` для escape-кодов
4. [ ] Добавить функции (git branch, etc.) если нужно
5. [ ] Сохранить файл
6. [ ] Выполнить `source ~/.bashrc`
7. [ ] Проверить работу в новом терминале
8. [ ] Проверить перенос длинных строк

## Резюме

- Сохраняйте PS1 в `~/.bashrc`
- Используйте переменные для цветов (удобнее читать и менять)
- Оборачивайте escape-коды в `\[` и `\]`
- Делайте бэкап перед изменениями
- Применяйте изменения через `source ~/.bashrc`
- Проверяйте синтаксис: `bash -n ~/.bashrc`
- Для сложных промптов используйте функции и PROMPT_COMMAND
