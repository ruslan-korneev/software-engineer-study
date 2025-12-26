# Tilde Expansion (раскрытие тильды)

## Что такое tilde expansion?

**Tilde expansion** — механизм shell, который заменяет `~` на путь к домашней директории.

```bash
$ echo ~
/home/username
```

## Основные формы

### ~ — домашняя директория текущего пользователя

```bash
$ echo ~
/home/user

$ cd ~
$ pwd
/home/user

$ ls ~/documents
# Содержимое /home/user/documents
```

### ~username — домашняя директория указанного пользователя

```bash
$ echo ~root
/root

$ echo ~www-data
/var/www

$ ls ~otheruser/public
# Содержимое /home/otheruser/public (если есть права)
```

### ~+ — текущая рабочая директория ($PWD)

```bash
$ cd /var/log
$ echo ~+
/var/log
# Эквивалент $PWD
```

### ~- — предыдущая рабочая директория ($OLDPWD)

```bash
$ cd /var/log
$ cd /etc
$ echo ~-
/var/log
# Эквивалент $OLDPWD
```

## Практические примеры

### Навигация
```bash
$ cd ~                   # в домашнюю директорию
$ cd ~/projects          # в ~/projects
$ cd ~-                  # в предыдущую директорию
```

### Работа с файлами
```bash
$ cp file.txt ~          # скопировать в домашнюю
$ mv ~/downloads/*.zip ~/archive/
$ cat ~/.bashrc          # файл конфигурации bash
```

### Создание путей
```bash
$ mkdir -p ~/projects/myapp
$ touch ~/notes.txt
```

## Когда tilde НЕ раскрывается

### В середине слова
```bash
$ echo foo~bar
foo~bar                  # НЕ раскрывается
```

### В кавычках
```bash
$ echo "~"
~                        # НЕ раскрывается

$ echo '~'
~                        # НЕ раскрывается
```

### Когда пользователь не существует
```bash
$ echo ~nonexistent
~nonexistent             # остаётся как есть
```

## Tilde в переменных

Tilde раскрывается при присваивании только в начале значения или после `:`:

```bash
$ PATH=~/bin:$PATH       # ~ раскроется
$ PATH=/usr/bin:~/bin    # ~ раскроется (после :)
$ VAR="~/test"           # ~ НЕ раскроется (в кавычках)
$ VAR=test~              # ~ НЕ раскроется (не в начале)
```

## Различия с $HOME

`~` и `$HOME` обычно эквивалентны, но есть нюансы:

```bash
$ echo ~
/home/user

$ echo $HOME
/home/user
```

Различия:
```bash
# ~ НЕ раскрывается в кавычках
$ echo "~"
~
$ echo "$HOME"
/home/user

# ~ раскрывается до подстановки переменных
$ unset HOME
$ echo ~
/home/user              # берётся из /etc/passwd
$ echo $HOME
                        # пусто
```

## Использование в скриптах

### Рекомендуется: $HOME
```bash
#!/bin/bash
CONFIG="$HOME/.config/myapp/config.yml"
# $HOME работает в кавычках, надёжнее
```

### Можно: ~
```bash
#!/bin/bash
cd ~
CONFIG=~/.config/myapp/config.yml
# ~ работает, но не в кавычках
```

## Tilde с числами (в bash)

Для работы со стеком директорий `pushd`/`popd`:

```bash
$ pushd /var/log
$ pushd /etc
$ pushd /tmp

$ dirs
/tmp /etc /var/log ~

$ echo ~0
/tmp                     # первая в стеке

$ echo ~1
/etc                     # вторая в стеке

$ echo ~2
/var/log                 # третья в стеке

$ cd ~1                  # перейти к /etc
```

## Примеры с путями

### Конфигурационные файлы
```bash
$ cat ~/.bashrc          # настройки bash
$ vim ~/.vimrc           # настройки vim
$ cat ~/.ssh/config      # SSH конфигурация
```

### Типичные директории
```bash
$ ls ~/.config/          # XDG конфигурации
$ ls ~/.local/bin/       # локальные исполняемые файлы
$ ls ~/.cache/           # кэш приложений
```

### Создание структуры
```bash
$ mkdir -p ~/projects/{web,mobile,scripts}
$ mkdir -p ~/.local/{bin,share,lib}
```

## Проверка раскрытия

```bash
$ echo ~/documents
/home/user/documents

$ echo "~/documents"
~/documents              # в кавычках не раскрывается

$ VAR=~/documents
$ echo $VAR
/home/user/documents     # раскрылось при присваивании
```

## Важные замечания

1. **Используйте $HOME в скриптах** для надёжности в кавычках
2. **~ должна быть в начале** слова или после `:`
3. **~ не работает в двойных кавычках** — используйте $HOME
4. **~username** требует существующего пользователя
5. **~+ и ~-** полезны для навигации между директориями
