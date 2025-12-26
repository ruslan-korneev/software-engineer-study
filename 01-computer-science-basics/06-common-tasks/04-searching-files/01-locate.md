# locate - быстрый поиск файлов по базе данных

## Что такое locate?

**locate** - утилита для быстрого поиска файлов по имени. В отличие от `find`, который сканирует файловую систему в реальном времени, `locate` использует предварительно созданную базу данных, что делает поиск мгновенным.

## Установка

```bash
# Debian/Ubuntu
sudo apt install mlocate
# или
sudo apt install plocate    # современная версия

# RHEL/CentOS
sudo yum install mlocate

# Fedora
sudo dnf install mlocate

# macOS (с Homebrew)
brew install findutils
# используется glocate
```

## База данных locate

### Обновление базы

База данных обычно обновляется автоматически (ежедневно через cron), но можно обновить вручную:

```bash
# Обновить базу данных
sudo updatedb

# Показать статистику базы
locate -S
# или
locate --statistics

# Пример вывода:
# Database /var/lib/mlocate/mlocate.db:
#     8,123 directories
#    82,456 files
#  4,567,890 bytes in file names
# 15,234,567 bytes used to store database
```

### Расположение базы данных

```bash
# Стандартное расположение
/var/lib/mlocate/mlocate.db

# plocate использует
/var/lib/plocate/plocate.db

# Конфигурация updatedb
/etc/updatedb.conf
```

### Конфигурация /etc/updatedb.conf

```bash
# Исключаемые файловые системы
PRUNEFS="NFS nfs nfs4 rpc_pipefs afs binfmt_misc"

# Исключаемые пути
PRUNEPATHS="/tmp /var/tmp /var/cache /var/spool"

# Исключаемые имена директорий
PRUNENAMES=".git .svn .hg"

# Индексировать bind-mounted директории
PRUNE_BIND_MOUNTS="yes"
```

## Базовое использование

```bash
# Найти файлы по имени
locate filename

# Найти все файлы .conf
locate .conf

# Найти файлы с путём
locate /etc/nginx

# Примеры:
locate nginx.conf
# /etc/nginx/nginx.conf
# /usr/share/doc/nginx/examples/nginx.conf

locate python
# /usr/bin/python
# /usr/bin/python3
# /usr/lib/python3.10/...
```

## Основные опции

```bash
# Игнорировать регистр
locate -i README
locate --ignore-case readme

# Ограничить количество результатов
locate -n 10 python
locate --limit 10 python

# Подсчитать количество совпадений
locate -c python
locate --count python

# Показать только существующие файлы
locate -e filename
locate --existing filename

# Базовое имя (без пути)
locate -b '\filename'
locate --basename '\filename'

# Тихий режим (только код возврата)
locate -q filename

# Использовать регулярные выражения
locate -r '\.conf$'
locate --regex '\.py$'
```

## Поиск по базовому имени

По умолчанию locate ищет подстроку в полном пути. Для поиска только по имени файла:

```bash
# Найдёт /path/to/config/file и /path/file/config
locate config

# Найдёт только файлы с именем config
locate -b '\config'

# Эквивалентно
locate --basename '\config'
```

## Регулярные выражения

```bash
# Найти файлы .conf в /etc
locate -r '^/etc/.*\.conf$'

# Найти файлы, начинающиеся с python
locate -r '/python[0-9]*$'

# Найти .py файлы
locate -r '\.py$'

# Найти файлы с цифрами в имени
locate -r '/[0-9]+\.log$'
```

## Альтернативы locate

### mlocate

Более новая версия locate с улучшенной безопасностью (учитывает права доступа):

```bash
# Установка
sudo apt install mlocate

# Использование такое же
locate filename
```

### plocate

Самая современная реализация, быстрее и эффективнее:

```bash
# Установка
sudo apt install plocate

# Использование такое же
locate filename

# Особенности:
# - Меньший размер базы
# - Быстрее поиск
# - Меньше памяти
```

### Сравнение

| Особенность | locate | mlocate | plocate |
|-------------|--------|---------|---------|
| Права доступа | Нет | Да | Да |
| Скорость | Быстро | Быстро | Очень быстро |
| Размер БД | Большой | Средний | Маленький |
| Активно развивается | Нет | Нет | Да |

## which, whereis, type

### which - найти исполняемый файл

```bash
# Найти путь к команде
which python
# /usr/bin/python

which -a python    # все варианты в PATH
# /usr/bin/python
# /usr/local/bin/python

# which ищет только в PATH
echo $PATH
```

### whereis - найти бинарники, исходники, man-страницы

```bash
# Найти всё связанное с командой
whereis python
# python: /usr/bin/python3.10 /usr/lib/python3.10 /usr/share/man/man1/python.1.gz

# Только бинарники
whereis -b python

# Только man-страницы
whereis -m python

# Только исходники
whereis -s gcc
```

### type - информация о команде

```bash
# Показать тип команды
type ls
# ls is aliased to 'ls --color=auto'

type cd
# cd is a shell builtin

type python
# python is /usr/bin/python

# Все варианты
type -a ls

# Только тип (alias, builtin, file, keyword, function)
type -t ls
# alias
```

## Практические примеры

### Найти все конфигурационные файлы nginx

```bash
locate -i nginx | grep -E '\.conf$'
```

### Быстро найти лог-файлы

```bash
locate -r '/var/log/.*\.log$' | head -20
```

### Найти все скрипты Python пользователя

```bash
locate -r "^/home/$(whoami)/.*\.py$"
```

### Проверить существование файла по паттерну

```bash
if locate -q -l 1 'important.conf'; then
    echo "Файл найден"
else
    echo "Файл не найден"
fi
```

### Комбинация с другими командами

```bash
# Найти и открыть в редакторе
vim $(locate -n 1 nginx.conf)

# Найти и показать размер
locate '*.log' | xargs ls -lh 2>/dev/null | head

# Найти и подсчитать строки
locate -r '\.py$' | head -10 | xargs wc -l
```

## Ограничения locate

### 1. Устаревшие данные

База данных может быть устаревшей:

```bash
# Создать файл
touch /tmp/newfile.txt

# Сразу не найдётся
locate newfile.txt
# ничего

# Обновить базу
sudo updatedb

# Теперь найдётся
locate newfile.txt
# /tmp/newfile.txt
```

### 2. Права доступа

- **locate** - показывает все файлы, даже недоступные
- **mlocate/plocate** - учитывают права доступа

### 3. Исключённые директории

Некоторые директории не индексируются (см. /etc/updatedb.conf).

## Когда использовать locate vs find

| Ситуация | Использовать |
|----------|--------------|
| Быстрый поиск по имени | `locate` |
| Нужны свежие данные | `find` |
| Поиск по атрибутам (размер, время) | `find` |
| Выполнить действие над файлами | `find` |
| Поиск в конкретной директории | `find` |
| Поиск везде | `locate` |

## Резюме команд

| Команда | Описание |
|---------|----------|
| `locate filename` | Найти файлы по имени |
| `locate -i name` | Без учёта регистра |
| `locate -n 10 name` | Ограничить результаты |
| `locate -c name` | Подсчитать совпадения |
| `locate -r 'regex'` | Поиск по регулярке |
| `locate -b '\name'` | Только по имени файла |
| `locate -e name` | Только существующие |
| `sudo updatedb` | Обновить базу |
| `locate -S` | Статистика базы |
| `which cmd` | Путь к команде |
| `whereis cmd` | Бинарники + man + src |
| `type cmd` | Информация о команде |
