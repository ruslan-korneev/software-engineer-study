# Автодополнение (Tab Completion)

[prev: 02-text-editing](./02-text-editing.md) | [next: 04-history](./04-history.md)

---
## Что такое автодополнение?

**Автодополнение** — функция shell, которая завершает частично введённые команды, пути к файлам и аргументы при нажатии клавиши Tab.

## Базовое использование

### Дополнение команд
```bash
$ whi<Tab>
$ which           # дополнено

$ pyt<Tab>
$ python3         # если единственный вариант
```

### Дополнение файлов
```bash
$ cat /etc/hos<Tab>
$ cat /etc/hosts

$ ls /ho<Tab>
$ ls /home/
```

### Дополнение директорий
```bash
$ cd /va<Tab>
$ cd /var/

$ cd ~/Doc<Tab>
$ cd ~/Documents/
```

## Одиночное и двойное нажатие Tab

### Одиночное нажатие Tab
- Если единственный вариант — дополняет
- Если несколько — дополняет общую часть

```bash
$ cd /etc/sys<Tab>
$ cd /etc/sysctl.     # общая часть дополнена
```

### Двойное нажатие Tab (Tab Tab)
- Показывает все возможные варианты

```bash
$ cd /etc/sys<Tab><Tab>
sysctl.conf  sysctl.d/  systemd/
```

## Дополнение разных элементов

### Команды
```bash
$ git<Tab><Tab>
git            git-receive-pack    gitk
git-cvsserver  git-shell          gitweb
git-lfs        git-upload-archive
git-mediawiki  git-upload-pack
```

### Переменные окружения
```bash
$ echo $HOM<Tab>
$ echo $HOME

$ echo $PA<Tab><Tab>
$PATH  $PAGER  $PAPERSIZE
```

### Имена пользователей (после ~)
```bash
$ cd ~ro<Tab>
$ cd ~root/
```

### Имена хостов (после @)
```bash
$ ssh user@local<Tab>
$ ssh user@localhost
```

## Программируемое дополнение

Bash поддерживает контекстное дополнение — разные варианты для разных команд.

### Git
```bash
$ git ch<Tab><Tab>
checkout  cherry  cherry-pick

$ git checkout ma<Tab>
$ git checkout main

$ git add <Tab><Tab>
# показывает изменённые файлы
```

### SSH
```bash
$ ssh <Tab><Tab>
# показывает хосты из ~/.ssh/config и known_hosts
```

### Docker
```bash
$ docker <Tab><Tab>
attach  build   commit  ...

$ docker stop <Tab><Tab>
# показывает запущенные контейнеры
```

## Настройка автодополнения

### Включение в bash
```bash
# В ~/.bashrc
if [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
fi
```

### Установка дополнительных пакетов
```bash
# Ubuntu/Debian
$ sudo apt install bash-completion

# macOS (с Homebrew)
$ brew install bash-completion@2
```

### Настройка регистронезависимости
```bash
# В ~/.inputrc
set completion-ignore-case on
```

### Показывать все при первом Tab
```bash
# В ~/.inputrc
set show-all-if-ambiguous on
```

## Опции дополнения

```bash
# Показать дополнения с цветом
set colored-stats on

# Показать тип файла (/ для директорий и т.д.)
set visible-stats on

# Дополнять до уникального префикса сразу
set show-all-if-unmodified on

# Игнорировать регистр
set completion-ignore-case on
```

Добавьте в `~/.inputrc` для постоянных настроек.

## Создание своих дополнений

### Простой пример
```bash
# Дополнение для команды myapp
complete -W "start stop restart status" myapp

$ myapp <Tab><Tab>
start  stop  restart  status
```

### Дополнение файлами определённого типа
```bash
# Только .txt файлы для команды edit
complete -f -X '!*.txt' edit
```

### Использование функции
```bash
_my_completion() {
    local cur=${COMP_WORDS[COMP_CWORD]}
    COMPREPLY=( $(compgen -W "option1 option2 option3" -- "$cur") )
}
complete -F _my_completion mycommand
```

## Дополнение в Zsh

Zsh имеет ещё более мощное автодополнение:

```bash
# Включить
autoload -Uz compinit && compinit

# Особенности:
# - Меню выбора (можно выбирать стрелками)
# - Описания опций
# - Исправление опечаток
# - Fuzzy matching
```

## Полезные комбинации

### Alt + ? — список всех дополнений
```bash
$ git <Alt+?>
# Показывает все варианты с описаниями
```

### Alt + * — вставить все варианты
```bash
$ ls *.txt<Alt+*>
$ ls file1.txt file2.txt file3.txt
# Все .txt файлы вставлены
```

## Советы

1. **Нажимайте Tab постоянно** — это экономит время и предотвращает ошибки
2. **Двойной Tab** показывает варианты
3. **Установите bash-completion** для контекстного дополнения
4. **Настройте `.inputrc`** под себя
5. **В zsh** дополнение ещё мощнее — рассмотрите переход

## Частые проблемы

### Tab не работает
```bash
# Проверьте, что bash-completion загружен
$ complete -p | head

# Загрузите вручную
$ source /etc/bash_completion
```

### Медленное дополнение
```bash
# Отключите затратные дополнения
# Или используйте zsh с async completion
```

### Нет дополнения для новой команды
```bash
# Посмотрите доступные файлы дополнений
$ ls /usr/share/bash-completion/completions/
```

---

[prev: 02-text-editing](./02-text-editing.md) | [next: 04-history](./04-history.md)
