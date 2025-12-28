# Получение справки: help, man, info

[prev: 01-type-which](./01-type-which.md) | [next: 03-alias](./03-alias.md)

---
## Обзор систем справки

В Unix/Linux есть несколько способов получить справку:

| Команда | Назначение |
|---------|------------|
| `--help` | Краткая справка от программы |
| `help` | Справка по встроенным командам shell |
| `man` | Традиционные man-страницы |
| `info` | Расширенная документация GNU |
| `apropos` | Поиск по описаниям man-страниц |

## Опция --help

Большинство программ поддерживают `--help`:

```bash
$ ls --help
Usage: ls [OPTION]... [FILE]...
List information about the FILEs...

$ grep --help
Usage: grep [OPTION]... PATTERN [FILE]...
Search for PATTERN in each FILE...
```

Иногда используется `-h`:
```bash
$ python -h
```

## Команда help

**help** — справка по встроенным командам bash:

```bash
$ help cd
cd: cd [-L|[-P [-e]] [-@]] [dir]
    Change the shell working directory...

$ help for
for: for NAME [in WORDS ... ] ; do COMMANDS; done
    Execute commands for each member in a list...
```

### Список всех builtins
```bash
$ help               # все встроенные команды
$ help -s cd         # короткий синтаксис
$ help -d cd         # краткое описание
```

## Команда man

**man** (manual) — основная система документации Unix.

### Базовое использование
```bash
$ man ls
$ man grep
$ man bash
```

### Навигация в man
| Клавиша | Действие |
|---------|----------|
| `Space` | Страница вниз |
| `b` | Страница вверх |
| `/pattern` | Поиск |
| `n` | Следующее совпадение |
| `N` | Предыдущее совпадение |
| `q` | Выход |
| `h` | Справка |

### Секции man-страниц

Man-страницы разделены на секции:

| Секция | Содержимое |
|--------|------------|
| 1 | Пользовательские команды |
| 2 | Системные вызовы |
| 3 | Библиотечные функции |
| 4 | Специальные файлы (/dev) |
| 5 | Форматы файлов и конфигурации |
| 6 | Игры |
| 7 | Разное |
| 8 | Системное администрирование |

### Указание секции
```bash
$ man passwd          # первая найденная (обычно секция 1)
$ man 1 passwd        # команда passwd
$ man 5 passwd        # формат файла /etc/passwd
```

### Поиск man-страниц

```bash
$ man -k copy         # поиск по ключевому слову
cp (1)               - copy files and directories
dd (1)               - convert and copy a file

$ apropos copy        # то же самое

$ man -f passwd       # краткое описание
passwd (1)           - change user password
passwd (5)           - the password file
```

### Структура man-страницы

Типичная man-страница содержит:

```
NAME        — имя и краткое описание
SYNOPSIS    — синтаксис использования
DESCRIPTION — подробное описание
OPTIONS     — опции командной строки
EXAMPLES    — примеры использования
FILES       — связанные файлы
SEE ALSO    — связанные страницы
BUGS        — известные проблемы
AUTHOR      — автор
```

## Команда info

**info** — система документации GNU с гипертекстовыми ссылками.

```bash
$ info coreutils      # документация GNU coreutils
$ info bash           # полная документация bash
```

### Навигация в info
| Клавиша | Действие |
|---------|----------|
| `Space` | Страница вниз |
| `Del` | Страница вверх |
| `n` | Следующий узел |
| `p` | Предыдущий узел |
| `u` | Вверх по иерархии |
| `Enter` | Перейти по ссылке |
| `l` | Назад |
| `q` | Выход |
| `h` | Справка |

## Другие источники справки

### whatis — краткое описание
```bash
$ whatis ls
ls (1) - list directory contents

$ whatis grep sed awk
grep (1) - print lines matching a pattern
sed (1) - stream editor for filtering and transforming text
awk (1) - pattern scanning and processing language
```

### Документация в /usr/share/doc
```bash
$ ls /usr/share/doc/bash
$ less /usr/share/doc/bash/README
```

### Онлайн ресурсы
- **tldr** — упрощённые man-страницы с примерами
  ```bash
  $ tldr tar
  ```
- **explainshell.com** — объяснение команд онлайн
- **man7.org** — онлайн man-страницы

## Практические примеры

### Поиск нужной команды
```bash
$ man -k "search file"
find (1) - search for files in a directory hierarchy
locate (1) - find files by name
```

### Узнать все опции команды
```bash
$ man ls | grep -A2 "^\s*-l"
       -l     use a long listing format
```

### Быстрый просмотр синтаксиса
```bash
$ man -h tar | head -20
# или
$ tar --help | head
```

### Сохранить man-страницу
```bash
$ man ls > ls-manual.txt           # текст
$ man -t ls > ls-manual.ps         # PostScript
$ man -t ls | ps2pdf - ls.pdf      # PDF
```

## Советы

1. **Начинайте с `--help`** — самый быстрый способ
2. **Используйте `man`** для подробной информации
3. **`man -k`** или `apropos` для поиска команд
4. **Читайте секцию EXAMPLES** — обычно самая полезная
5. **Установите tldr** для быстрых примеров:
   ```bash
   $ npm install -g tldr
   $ tldr tar
   ```

---

[prev: 01-type-which](./01-type-which.md) | [next: 03-alias](./03-alias.md)
