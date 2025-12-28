# find - базовый поиск файлов

[prev: 01-locate](./01-locate.md) | [next: 03-find-advanced](./03-find-advanced.md)

---
## Что такое find?

**find** - мощная утилита для поиска файлов и директорий в реальном времени. В отличие от `locate`, find сканирует файловую систему непосредственно при запуске, что гарантирует актуальные результаты.

## Базовый синтаксис

```bash
find [путь...] [выражение]
```

- **путь** - где искать (по умолчанию текущая директория)
- **выражение** - условия поиска и действия

## Поиск по имени

### Простой поиск

```bash
# Найти файлы по точному имени
find /home -name "file.txt"

# Поиск в текущей директории
find . -name "*.py"

# Несколько путей
find /etc /usr -name "*.conf"

# Игнорировать регистр
find . -iname "readme.md"
find . -iname "README*"
```

### Шаблоны (glob patterns)

```bash
# * - любые символы
find . -name "*.txt"
find . -name "test*"
find . -name "*config*"

# ? - один символ
find . -name "file?.txt"     # file1.txt, fileA.txt

# [...] - один символ из набора
find . -name "file[123].txt" # file1.txt, file2.txt, file3.txt
find . -name "file[a-z].txt" # filea.txt, fileb.txt...

# Важно: кавычки обязательны для защиты от shell expansion
find . -name "*.log"    # правильно
find . -name *.log      # неправильно - shell развернёт раньше find
```

### Поиск по пути

```bash
# -path - шаблон для полного пути
find . -path "*/test/*"

# -wholename - синоним -path
find . -wholename "*config*"

# Игнорировать регистр
find . -ipath "*test*"
```

## Поиск по типу

```bash
# -type определяет тип объекта
find . -type f    # обычные файлы
find . -type d    # директории
find . -type l    # символические ссылки
find . -type b    # блочные устройства
find . -type c    # символьные устройства
find . -type p    # именованные каналы (FIFO)
find . -type s    # сокеты

# Примеры
find /etc -type f -name "*.conf"     # только файлы .conf
find . -type d -name "test"          # только директории с именем test
find . -type l                        # все символические ссылки
```

## Поиск по размеру

```bash
# -size [+-]n[cwbkMG]
# + больше, - меньше, без знака - точно
# c - байты, w - слова (2 байта), b - блоки 512 байт (по умолчанию)
# k - килобайты, M - мегабайты, G - гигабайты

# Файлы больше 100 МБ
find . -size +100M

# Файлы меньше 1 КБ
find . -size -1k

# Файлы точно 1024 байта
find . -size 1024c

# Пустые файлы
find . -size 0
find . -empty    # пустые файлы и директории

# Диапазон размеров
find . -size +10M -size -100M
```

## Поиск по времени

### Типы временных меток

- **mtime** - время изменения содержимого (modification time)
- **atime** - время доступа (access time)
- **ctime** - время изменения метаданных (change time - права, владелец)

### Поиск по дням

```bash
# Файлы изменённые за последние 7 дней
find . -mtime -7

# Файлы изменённые более 30 дней назад
find . -mtime +30

# Файлы изменённые ровно 1 день назад (24-48 часов)
find . -mtime 1

# По времени доступа
find . -atime -7

# По времени изменения метаданных
find . -ctime -7
```

### Поиск по минутам

```bash
# Файлы изменённые за последние 60 минут
find . -mmin -60

# Файлы не открывавшиеся более 30 минут
find . -amin +30
```

### Сравнение с файлом

```bash
# Файлы новее указанного файла
find . -newer reference.txt

# Файлы старше указанного файла
find . ! -newer reference.txt
find . -not -newer reference.txt

# Создать файл-маркер и искать относительно него
touch -d "2024-01-01" /tmp/marker
find . -newer /tmp/marker
```

## Поиск по владельцу и правам

### По владельцу

```bash
# Файлы пользователя
find . -user username
find . -user root

# По UID
find . -uid 1000

# Файлы группы
find . -group developers
find . -gid 100

# Файлы без владельца (удалённый пользователь)
find . -nouser
find . -nogroup
```

### По правам доступа

```bash
# Точное соответствие прав
find . -perm 644        # точно rw-r--r--
find . -perm u=rw,g=r,o=r  # то же в символьной форме

# Все указанные биты установлены (AND)
find . -perm -644       # как минимум rw-r--r--
find . -perm -u=rw      # владелец имеет rw

# Любой из указанных битов установлен (OR)
find . -perm /644       # любой из битов rw-r--r--
find . -perm /u=x,g=x,o=x  # исполняемый кем-либо

# Практические примеры
find . -perm -u=s       # файлы с SUID
find . -perm -g=s       # файлы с SGID
find . -perm -o=w       # файлы доступные для записи всем
find . -perm -111       # исполняемые всеми
```

## Логические операторы

```bash
# AND (неявно или -a)
find . -name "*.py" -type f
find . -name "*.py" -a -type f    # явный AND

# OR (-o)
find . -name "*.py" -o -name "*.js"
find . \( -name "*.py" -o -name "*.js" \) -type f

# NOT (! или -not)
find . ! -name "*.txt"
find . -not -name "*.txt"
find . -type f ! -name "*.log"

# Группировка (экранированные скобки)
find . \( -name "*.py" -o -name "*.js" \) -a -type f
find . -type f \( -name "*.py" -o -name "*.js" \)
```

## Ограничение глубины поиска

```bash
# Максимальная глубина
find . -maxdepth 1 -name "*.txt"    # только текущая директория
find . -maxdepth 2 -name "*.txt"    # текущая + 1 уровень вниз

# Минимальная глубина
find . -mindepth 2 -name "*.txt"    # начиная со 2 уровня

# Комбинация
find . -mindepth 2 -maxdepth 4 -name "*.txt"
```

## Исключение директорий

```bash
# Исключить директорию .git
find . -path ./.git -prune -o -name "*.py" -print

# Исключить несколько директорий
find . \( -path ./node_modules -o -path ./.git \) -prune -o -name "*.js" -print

# Исключить по имени
find . -name "node_modules" -prune -o -name "*.js" -print

# Более простой синтаксис (не заходить внутрь)
find . -not -path "*/node_modules/*" -name "*.js"
find . ! -path "*/.git/*" -name "*.py"
```

## Практические примеры

### Поиск конфигурационных файлов

```bash
# Все .conf файлы в /etc
find /etc -type f -name "*.conf"

# Файлы изменённые за последнюю неделю
find /etc -type f -name "*.conf" -mtime -7
```

### Поиск больших файлов

```bash
# Файлы больше 100 МБ
find /var -type f -size +100M 2>/dev/null

# Топ-10 больших файлов
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -h | tail -10
```

### Поиск устаревших файлов

```bash
# Файлы не изменявшиеся более года
find /tmp -type f -mtime +365

# Старые логи
find /var/log -name "*.log" -mtime +30
```

### Поиск по содержимому (комбинация с grep)

```bash
# Найти файлы содержащие текст
find . -type f -name "*.py" -exec grep -l "import os" {} \;

# С показом совпадений
find . -type f -name "*.py" -exec grep -H "import os" {} \;
```

### Поиск исполняемых файлов

```bash
# Все исполняемые файлы пользователя
find ~/bin -type f -perm -u=x

# Скрипты
find . -type f -name "*.sh" -perm -u=x
```

### Безопасность: поиск проблемных прав

```bash
# Файлы с SUID битом
find / -type f -perm -4000 2>/dev/null

# Файлы доступные для записи всем
find / -type f -perm -0002 2>/dev/null

# Файлы без владельца
find / -nouser -o -nogroup 2>/dev/null
```

## Ошибки и их подавление

```bash
# Перенаправить ошибки (Permission denied)
find / -name "*.conf" 2>/dev/null

# Или отдельно ошибки и результат
find / -name "*.conf" 2>errors.log

# Показать только ошибки
find / -name "*.conf" 2>&1 >/dev/null
```

## Резюме опций

| Опция | Описание |
|-------|----------|
| `-name "pattern"` | Поиск по имени (регистрозависимый) |
| `-iname "pattern"` | Поиск по имени (без регистра) |
| `-type f/d/l` | Тип: файл/директория/ссылка |
| `-size [+-]n[kMG]` | По размеру |
| `-mtime [+-]n` | По времени изменения (дни) |
| `-mmin [+-]n` | По времени изменения (минуты) |
| `-user name` | По владельцу |
| `-group name` | По группе |
| `-perm mode` | По правам |
| `-maxdepth n` | Максимальная глубина |
| `-mindepth n` | Минимальная глубина |
| `-empty` | Пустые файлы/директории |
| `-newer file` | Новее указанного файла |
| `-a` | AND (по умолчанию) |
| `-o` | OR |
| `!` или `-not` | NOT |

---

[prev: 01-locate](./01-locate.md) | [next: 03-find-advanced](./03-find-advanced.md)
