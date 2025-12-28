# Parameter Expansion (раскрытие параметров)

[prev: 03-brace-expansion](./03-brace-expansion.md) | [next: 05-command-substitution](./05-command-substitution.md)

---
## Что такое parameter expansion?

**Parameter expansion** — механизм получения и преобразования значений переменных shell.

```bash
$ NAME="World"
$ echo $NAME
World
$ echo ${NAME}
World
```

## Базовый синтаксис

### Простое раскрытие
```bash
$ VAR="value"
$ echo $VAR          # короткая форма
$ echo ${VAR}        # полная форма
```

### Когда нужны фигурные скобки
```bash
$ FILE="report"
$ echo $FILE_2024    # ищет переменную FILE_2024
$ echo ${FILE}_2024  # правильно: report_2024
```

## Значения по умолчанию

### ${var:-default} — значение по умолчанию
```bash
$ unset NAME
$ echo ${NAME:-Anonymous}
Anonymous

$ NAME="Alice"
$ echo ${NAME:-Anonymous}
Alice
```

### ${var:=default} — присвоить если не задано
```bash
$ unset NAME
$ echo ${NAME:=Anonymous}
Anonymous
$ echo $NAME         # теперь NAME=Anonymous
```

### ${var:+alternative} — альтернатива если задано
```bash
$ NAME="Alice"
$ echo ${NAME:+Hello, $NAME}
Hello, Alice

$ unset NAME
$ echo ${NAME:+Hello, $NAME}
                     # пусто
```

### ${var:?message} — ошибка если не задано
```bash
$ unset NAME
$ echo ${NAME:?Variable not set}
bash: NAME: Variable not set
```

## Работа со строками

### Длина строки
```bash
$ STR="Hello, World!"
$ echo ${#STR}
13
```

### Извлечение подстроки
```bash
$ STR="Hello, World!"
$ echo ${STR:0:5}    # с позиции 0, 5 символов
Hello

$ echo ${STR:7}      # с позиции 7 до конца
World!

$ echo ${STR: -6}    # последние 6 символов (пробел важен!)
World!
```

## Удаление по шаблону

### Удаление с начала
```bash
$ FILE="/home/user/documents/report.txt"

# Самое короткое совпадение
$ echo ${FILE#*/}
home/user/documents/report.txt

# Самое длинное совпадение
$ echo ${FILE##*/}
report.txt          # только имя файла
```

### Удаление с конца
```bash
$ FILE="/home/user/documents/report.txt"

# Самое короткое совпадение
$ echo ${FILE%.*}
/home/user/documents/report

# Самое длинное совпадение
$ echo ${FILE%%/*}
                    # пусто (удалено всё после первого /)
```

### Практические примеры
```bash
# Получить имя файла без пути
$ FILE="/path/to/file.txt"
$ echo ${FILE##*/}
file.txt

# Получить расширение
$ echo ${FILE##*.}
txt

# Получить путь без файла
$ echo ${FILE%/*}
/path/to

# Удалить расширение
$ echo ${FILE%.*}
/path/to/file
```

## Замена подстрок

### Первое вхождение
```bash
$ STR="hello hello hello"
$ echo ${STR/hello/hi}
hi hello hello
```

### Все вхождения
```bash
$ echo ${STR//hello/hi}
hi hi hi
```

### Замена в начале
```bash
$ STR="hello world"
$ echo ${STR/#hello/hi}
hi world
```

### Замена в конце
```bash
$ echo ${STR/%world/everyone}
hello everyone
```

## Преобразование регистра (bash 4+)

### В верхний регистр
```bash
$ NAME="hello"
$ echo ${NAME^}      # первая буква
Hello

$ echo ${NAME^^}     # все буквы
HELLO
```

### В нижний регистр
```bash
$ NAME="HELLO"
$ echo ${NAME,}      # первая буква
hELLO

$ echo ${NAME,,}     # все буквы
hello
```

## Косвенное раскрытие

```bash
$ VAR="NAME"
$ NAME="Alice"
$ echo ${!VAR}       # получить значение переменной NAME
Alice
```

## Список переменных по префиксу

```bash
$ APP_NAME="MyApp"
$ APP_VERSION="1.0"
$ APP_PORT=8080

$ echo ${!APP_*}     # имена переменных с префиксом APP_
APP_NAME APP_PORT APP_VERSION
```

## Массивы

```bash
$ ARR=(one two three)

$ echo ${ARR[0]}     # первый элемент
one

$ echo ${ARR[@]}     # все элементы
one two three

$ echo ${#ARR[@]}    # количество элементов
3

$ echo ${ARR[@]:1:2} # срез: с позиции 1, 2 элемента
two three
```

## Практические примеры

### Обработка путей файлов
```bash
FILE="/home/user/project/src/main.py"

BASENAME=${FILE##*/}       # main.py
DIRNAME=${FILE%/*}         # /home/user/project/src
EXTENSION=${FILE##*.}      # py
FILENAME=${BASENAME%.*}    # main
```

### Значение по умолчанию для конфигурации
```bash
PORT=${PORT:-8080}
HOST=${HOST:-localhost}
DEBUG=${DEBUG:-false}
```

### Проверка обязательных переменных
```bash
: ${DB_HOST:?Database host not set}
: ${DB_USER:?Database user not set}
```

### Генерация имён файлов
```bash
BACKUP="${FILE%.*}-$(date +%Y%m%d).${FILE##*.}"
# main-20241226.py
```

## Сводная таблица

| Синтаксис | Описание |
|-----------|----------|
| `${var}` | Значение переменной |
| `${#var}` | Длина значения |
| `${var:-default}` | Default если не задано |
| `${var:=default}` | Присвоить если не задано |
| `${var:+alt}` | Альтернатива если задано |
| `${var:?error}` | Ошибка если не задано |
| `${var:offset:length}` | Подстрока |
| `${var#pattern}` | Удалить с начала (короткое) |
| `${var##pattern}` | Удалить с начала (длинное) |
| `${var%pattern}` | Удалить с конца (короткое) |
| `${var%%pattern}` | Удалить с конца (длинное) |
| `${var/old/new}` | Заменить первое |
| `${var//old/new}` | Заменить все |
| `${var^}` / `${var^^}` | В верхний регистр |
| `${var,}` / `${var,,}` | В нижний регистр |

---

[prev: 03-brace-expansion](./03-brace-expansion.md) | [next: 05-command-substitution](./05-command-substitution.md)
