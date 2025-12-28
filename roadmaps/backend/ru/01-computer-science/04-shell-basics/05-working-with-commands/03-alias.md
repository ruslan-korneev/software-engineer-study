# Алиасы (alias)

[prev: 02-help-man-info](./02-help-man-info.md) | [next: 01-stdin-stdout-stderr](../06-redirection/01-stdin-stdout-stderr.md)

---
## Что такое алиас?

**Алиас** (alias) — это псевдоним для команды или последовательности команд. Позволяет создавать сокращения для часто используемых команд.

## Синтаксис

```bash
alias имя='команда'
alias имя="команда с переменными"
```

## Базовое использование

### Создание алиаса
```bash
$ alias ll='ls -la'
$ ll
total 48
drwxr-xr-x  5 user group  4096 Dec 26 10:00 .
...
```

### Просмотр алиасов
```bash
$ alias              # все алиасы
$ alias ll           # конкретный алиас
alias ll='ls -la'
```

### Удаление алиаса
```bash
$ unalias ll
$ ll
bash: ll: command not found
```

## Примеры полезных алиасов

### Сокращения для ls
```bash
alias ll='ls -la'           # подробный список
alias la='ls -la'           # то же
alias l='ls -CF'            # компактный вид
alias lh='ls -lh'           # с размерами
alias lt='ls -lt'           # по времени
```

### Безопасные операции
```bash
alias rm='rm -i'            # спрашивать перед удалением
alias cp='cp -i'            # спрашивать перед перезаписью
alias mv='mv -i'            # спрашивать перед перезаписью
```

### Навигация
```bash
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias ~='cd ~'
alias home='cd ~'
```

### Полезные утилиты
```bash
alias h='history'
alias c='clear'
alias j='jobs -l'
alias ports='netstat -tulanp'
alias path='echo -e ${PATH//:/\\n}'
```

### Git алиасы
```bash
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline'
alias gd='git diff'
alias gco='git checkout'
```

### Docker алиасы
```bash
alias d='docker'
alias dc='docker-compose'
alias dps='docker ps'
alias di='docker images'
```

### Python алиасы
```bash
alias py='python3'
alias pip='pip3'
alias venv='python3 -m venv'
alias activate='source venv/bin/activate'
```

## Постоянные алиасы

Алиасы, созданные в терминале, исчезают после закрытия сессии. Для постоянных алиасов добавьте их в файл конфигурации shell:

### Для bash
```bash
# ~/.bashrc
alias ll='ls -la'
alias gs='git status'
```

### Для zsh
```bash
# ~/.zshrc
alias ll='ls -la'
alias gs='git status'
```

### Отдельный файл алиасов
```bash
# ~/.bash_aliases
alias ll='ls -la'
alias gs='git status'

# В ~/.bashrc добавить:
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
```

После редактирования нужно перезагрузить конфигурацию:
```bash
$ source ~/.bashrc
# или
$ . ~/.bashrc
```

## Алиасы с аргументами

Алиасы не поддерживают аргументы напрямую. Для этого используйте функции:

```bash
# Алиас не работает с аргументами
alias greet='echo Hello, $1'    # НЕ работает

# Используйте функцию
greet() {
    echo "Hello, $1!"
}
$ greet World
Hello, World!

# Или однострочно
mkcd() { mkdir -p "$1" && cd "$1"; }
$ mkcd newdir
```

## Экранирование алиасов

### Выполнить команду без алиаса
```bash
$ alias ls='ls --color=auto'

# Способы выполнить оригинальную ls:
$ \ls                    # обратный слэш
$ 'ls'                   # кавычки
$ command ls             # команда command
$ /bin/ls                # полный путь
```

## Проверка алиасов

### type — покажет алиас
```bash
$ type ls
ls is aliased to 'ls --color=auto'
```

### which — покажет путь (не алиас)
```bash
$ which ls
/bin/ls
```

## Сложные алиасы

### Алиас с несколькими командами
```bash
alias update='sudo apt update && sudo apt upgrade'
alias weather='curl wttr.in'
alias myip='curl ifconfig.me'
```

### Алиас с конвейером
```bash
alias psg='ps aux | grep -v grep | grep -i'
$ psg python    # найти процессы python
```

### Алиас с подстановкой
```bash
alias now='date +"%Y-%m-%d %H:%M:%S"'
$ now
2024-12-26 14:30:00
```

## Условные алиасы

```bash
# Использовать bat вместо cat, если установлен
if command -v bat &> /dev/null; then
    alias cat='bat'
fi

# Разные алиасы для разных систем
if [[ "$OSTYPE" == "darwin"* ]]; then
    alias ls='ls -G'    # macOS
else
    alias ls='ls --color=auto'  # Linux
fi
```

## Стандартные алиасы в системах

Многие дистрибутивы устанавливают алиасы по умолчанию:

```bash
$ alias
alias ls='ls --color=auto'
alias grep='grep --color=auto'
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
```

## Советы

1. **Используйте осмысленные имена** — `gs` для `git status`, а не `a1`
2. **Не переопределяйте системные команды** без причины
3. **Добавьте `-i` к опасным командам** (rm, cp, mv)
4. **Храните алиасы в отдельном файле** для организации
5. **Документируйте сложные алиасы** комментариями
6. **Используйте функции** для команд с аргументами

## Шаблон файла алиасов

```bash
# ~/.bash_aliases

# Навигация
alias ..='cd ..'
alias ...='cd ../..'

# Листинг
alias ll='ls -la'
alias la='ls -A'

# Безопасность
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Git
alias gs='git status'
alias ga='git add'
alias gc='git commit -m'
alias gp='git push'

# Утилиты
alias h='history'
alias c='clear'
```

---

[prev: 02-help-man-info](./02-help-man-info.md) | [next: 01-stdin-stdout-stderr](../06-redirection/01-stdin-stdout-stderr.md)
