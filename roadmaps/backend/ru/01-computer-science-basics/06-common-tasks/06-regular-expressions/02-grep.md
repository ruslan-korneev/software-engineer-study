# grep - поиск по шаблону

## Что такое grep?

**grep** (Global Regular Expression Print) - утилита для поиска строк, соответствующих шаблону. Это одна из самых используемых команд в Unix/Linux.

## Базовый синтаксис

```bash
grep [опции] pattern [файл...]
```

## Простой поиск

```bash
# Найти строки с "error"
grep 'error' logfile.txt

# Поиск в нескольких файлах
grep 'error' *.log

# Рекурсивный поиск
grep -r 'error' /var/log/

# Из stdin (pipe)
cat file.txt | grep 'pattern'
ps aux | grep nginx
```

## Основные опции

### Регистр

```bash
# Игнорировать регистр (-i)
grep -i 'error' log.txt    # Error, ERROR, error

# Точное совпадение регистра (по умолчанию)
grep 'Error' log.txt       # только Error
```

### Тип regex

```bash
# BRE - Basic Regular Expressions (по умолчанию)
grep 'ab\+' file.txt

# ERE - Extended Regular Expressions (-E или egrep)
grep -E 'ab+' file.txt
egrep 'ab+' file.txt

# PCRE - Perl Compatible (-P)
grep -P '\d+' file.txt

# Фиксированные строки (-F или fgrep)
grep -F '*.txt' file.txt   # ищет буквально "*.txt"
fgrep '*.txt' file.txt
```

### Вывод

```bash
# Показать номера строк (-n)
grep -n 'error' log.txt
# 42:Error occurred
# 57:Connection error

# Показать имя файла (-H, по умолчанию для нескольких файлов)
grep -H 'error' *.log
# app.log:Error message
# system.log:Error detected

# Не показывать имя файла (-h)
grep -h 'error' *.log

# Только имена файлов с совпадениями (-l)
grep -l 'error' *.log
# app.log
# system.log

# Только имена файлов БЕЗ совпадений (-L)
grep -L 'error' *.log

# Подсчитать количество совпадений (-c)
grep -c 'error' log.txt
# 42
```

### Контекст

```bash
# Строки после совпадения (-A)
grep -A 3 'error' log.txt    # 3 строки после

# Строки до совпадения (-B)
grep -B 3 'error' log.txt    # 3 строки до

# Строки до и после (-C)
grep -C 3 'error' log.txt    # 3 строки до и после

# Пример вывода с контекстом:
# --
# 2024-01-15 10:29:58 INFO: Processing
# 2024-01-15 10:29:59 ERROR: Failed to connect
# 2024-01-15 10:30:00 INFO: Retrying
# --
```

### Инверсия и целые слова

```bash
# Инверсия - строки БЕЗ pattern (-v)
grep -v 'error' log.txt

# Целое слово (-w)
grep -w 'error' log.txt      # не найдёт "errors" или "errored"

# Целая строка (-x)
grep -x 'exact line' file.txt  # строка должна полностью совпадать
```

### Рекурсивный поиск

```bash
# Рекурсивно (-r)
grep -r 'TODO' /project/

# Рекурсивно, следовать по симлинкам (-R)
grep -R 'TODO' /project/

# С include/exclude
grep -r --include="*.py" 'import' /project/
grep -r --exclude="*.log" 'error' /project/
grep -r --exclude-dir=node_modules 'import' /project/
```

### Бинарные файлы

```bash
# Игнорировать бинарные файлы
grep -I 'pattern' *

# Обрабатывать бинарные как текст
grep -a 'pattern' binary_file

# Показать совпадения без контекста для бинарных
grep -o 'pattern' binary_file
```

## Регулярные выражения в grep

### BRE (Basic)

```bash
# Любой символ
grep 'h.t' words.txt           # hat, hit, hot

# Начало/конец строки
grep '^Error' log.txt          # начинается с Error
grep 'done$' log.txt           # заканчивается на done

# Класс символов
grep '[aeiou]' words.txt       # гласные
grep '[^0-9]' data.txt         # не цифры

# Повторения (BRE - нужно экранировать)
grep 'go*d' words.txt          # gd, god, good, goood
grep 'go\+d' words.txt         # god, good, goood (не gd)
grep 'go\?d' words.txt         # gd или god
grep 'go\{2,4\}d' words.txt    # good, goood, gooood
```

### ERE (Extended)

```bash
# То же самое, но без экранирования
grep -E 'go+d' words.txt
grep -E 'go?d' words.txt
grep -E 'go{2,4}d' words.txt

# Альтернатива
grep -E 'cat|dog' animals.txt

# Группировка
grep -E '(ab)+' text.txt       # ab, abab, ababab
grep -E 'file(name)?\.txt' files.txt  # file.txt или filename.txt
```

### PCRE (Perl)

```bash
# Специальные классы
grep -P '\d+' numbers.txt      # цифры
grep -P '\w+' text.txt         # буквы, цифры, _
grep -P '\s+' text.txt         # пробельные символы

# Границы слов
grep -P '\berror\b' log.txt    # только слово error

# Lookahead/Lookbehind
grep -P '(?<=@)\w+' emails.txt  # домен после @
grep -P '\w+(?=@)' emails.txt   # имя до @

# Не жадные квантификаторы
grep -Po '<.*?>' html.txt       # минимальные совпадения тегов
```

## Практические примеры

### Поиск в логах

```bash
# Ошибки с временем
grep -E '^\d{4}-\d{2}-\d{2}.*ERROR' app.log

# IP адреса
grep -E '\b[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\b' access.log

# HTTP коды 4xx и 5xx
grep -E ' [45][0-9]{2} ' access.log

# Последние 100 ошибок
grep 'ERROR' app.log | tail -100
```

### Поиск в коде

```bash
# TODO и FIXME комментарии
grep -rn -E '(TODO|FIXME):?' --include="*.py" /project/

# Функции в Python
grep -E '^def \w+' *.py

# Импорты
grep -h '^import\|^from' *.py | sort | uniq

# Неиспользуемые переменные (простой поиск)
grep -rn 'variable_name' --include="*.py" /project/
```

### Поиск файлов конфигурации

```bash
# Закомментированные строки
grep '^#' config.conf

# Непустые незакомментированные строки
grep -v '^#\|^$' config.conf

# Определённый параметр
grep -E '^server_name\s*=' nginx.conf
```

### Анализ данных

```bash
# Email адреса
grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# URL
grep -Eo 'https?://[^ ]+' file.txt

# Телефоны (простой паттерн)
grep -E '\+?[0-9]{1,3}[- ]?[0-9]{3}[- ]?[0-9]{4}' contacts.txt
```

### Работа с процессами

```bash
# Найти процесс (исключая сам grep)
ps aux | grep '[n]ginx'

# Или проще
pgrep -a nginx

# Посмотреть открытые порты
netstat -tlnp | grep ':80\|:443'
ss -tlnp | grep -E ':80|:443'
```

## Комбинации с другими командами

```bash
# Подсчитать уникальные ошибки
grep 'ERROR' log.txt | sort | uniq -c | sort -rn

# Найти файлы и искать в них
find . -name "*.py" -exec grep -l 'import os' {} \;

# Или через xargs
find . -name "*.py" -print0 | xargs -0 grep -l 'import os'

# Заменить найденное (с sed)
grep -l 'old_text' *.txt | xargs sed -i 's/old_text/new_text/g'

# Статистика по дням из лога
grep 'ERROR' app.log | grep -oP '^\d{4}-\d{2}-\d{2}' | uniq -c
```

## Альтернативы grep

### ripgrep (rg)

Современный быстрый grep:

```bash
# Установка
sudo apt install ripgrep

# Использование (очень быстрый, .gitignore по умолчанию)
rg 'pattern' /path/

# Типы файлов
rg -t py 'import' /project/
```

### ag (The Silver Searcher)

```bash
# Установка
sudo apt install silversearcher-ag

# Использование
ag 'pattern' /path/
```

### ack

```bash
# Установка
sudo apt install ack

# Использование
ack 'pattern' /path/
```

## Производительность

```bash
# Фиксированные строки быстрее regex
grep -F 'exact string' file.txt

# Ограничить количество совпадений
grep -m 10 'pattern' huge.log

# Использовать LC_ALL=C для ASCII
LC_ALL=C grep 'pattern' file.txt
```

## Резюме опций

| Опция | Описание |
|-------|----------|
| `-i` | Игнорировать регистр |
| `-v` | Инверсия (НЕ содержит) |
| `-n` | Номера строк |
| `-c` | Подсчитать совпадения |
| `-l` | Только имена файлов |
| `-L` | Файлы БЕЗ совпадений |
| `-r` | Рекурсивно |
| `-w` | Целое слово |
| `-x` | Целая строка |
| `-E` | Extended regex |
| `-P` | Perl regex |
| `-F` | Фиксированные строки |
| `-o` | Только совпадения |
| `-A n` | n строк после |
| `-B n` | n строк до |
| `-C n` | n строк контекста |
| `-m n` | Максимум n совпадений |
| `-q` | Тихий режим (только код возврата) |
