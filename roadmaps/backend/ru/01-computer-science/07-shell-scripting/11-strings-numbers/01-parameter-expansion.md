# Раскрытие параметров (Parameter Expansion)

[prev: 02-for-c-style](../10-for-loop/02-for-c-style.md) | [next: 02-string-operations](./02-string-operations.md)

---
## Что такое Parameter Expansion

Parameter Expansion - это механизм bash для работы с переменными и их преобразования. Базовая форма `${parameter}` может быть расширена множеством модификаторов.

## Базовое раскрытие

```bash
name="World"
echo "Hello, ${name}!"  # Hello, World!

# Фигурные скобки обязательны для:
echo "${name}ling"      # Worldling (без {} было бы $nameling)
echo "${10}"            # Десятый позиционный параметр
```

## Значения по умолчанию

### ${var:-default} - использовать default

```bash
# Если var не определена или пуста
name=""
echo "${name:-Anonymous}"  # Anonymous
echo "$name"               # (пусто, var не изменилась)

unset name
echo "${name:-Anonymous}"  # Anonymous
```

### ${var:=default} - присвоить default

```bash
# Если var не определена или пуста, ПРИСВОИТЬ default
unset name
echo "${name:=Anonymous}"  # Anonymous
echo "$name"               # Anonymous (var изменилась)
```

### ${var:+alternative} - альтернативное значение

```bash
# Если var определена и не пуста, использовать alternative
name="John"
echo "${name:+Hello, $name}"  # Hello, John

unset name
echo "${name:+Hello, $name}"  # (пусто)
```

### ${var:?message} - ошибка если пусто

```bash
# Выход с ошибкой если var не определена или пуста
unset required_var
echo "${required_var:?Ошибка: переменная обязательна}"
# bash: required_var: Ошибка: переменная обязательна
```

## Разница : и без :

```bash
var=""  # Пустая, но определённая переменная

# С : - проверяет на пустоту И неопределённость
echo "${var:-default}"   # default

# Без : - проверяет только на неопределённость
echo "${var-default}"    # (пусто, var определена)
```

## Длина строки

```bash
string="Hello, World!"
echo "${#string}"  # 13

# Длина массива
arr=(one two three)
echo "${#arr[@]}"  # 3 (количество элементов)
echo "${#arr[0]}"  # 3 (длина первого элемента)
```

## Подстрока

### ${var:offset}

```bash
string="Hello, World!"

echo "${string:7}"     # World!
echo "${string:0:5}"   # Hello
echo "${string:7:5}"   # World
```

### Отрицательные индексы

```bash
string="Hello, World!"

# Отрицательный offset - от конца
echo "${string: -6}"     # World! (пробел перед минусом обязателен)
echo "${string: -6:5}"   # World
```

## Удаление паттерна

### ${var#pattern} - удалить с начала (короткое)

```bash
filename="/path/to/file.txt"

echo "${filename#*/}"      # path/to/file.txt (удалён первый /)
```

### ${var##pattern} - удалить с начала (длинное)

```bash
filename="/path/to/file.txt"

echo "${filename##*/}"     # file.txt (удалено всё до последнего /)
# Эквивалент basename
```

### ${var%pattern} - удалить с конца (короткое)

```bash
filename="/path/to/file.txt"

echo "${filename%/*}"      # /path/to (удалено после последнего /)
echo "${filename%.*}"      # /path/to/file (удалено расширение)
```

### ${var%%pattern} - удалить с конца (длинное)

```bash
filename="file.tar.gz"

echo "${filename%.*}"      # file.tar (короткое - первое совпадение)
echo "${filename%%.*}"     # file (длинное - максимальное)
```

### Примеры работы с путями

```bash
filepath="/home/user/documents/report.pdf"

# Имя файла (basename)
echo "${filepath##*/}"          # report.pdf

# Директория (dirname)
echo "${filepath%/*}"           # /home/user/documents

# Расширение
echo "${filepath##*.}"          # pdf

# Имя без расширения
filename="${filepath##*/}"
echo "${filename%.*}"           # report
```

## Замена паттерна

### ${var/pattern/replacement} - первое вхождение

```bash
text="hello hello hello"

echo "${text/hello/hi}"    # hi hello hello
```

### ${var//pattern/replacement} - все вхождения

```bash
text="hello hello hello"

echo "${text//hello/hi}"   # hi hi hi
```

### ${var/#pattern/replacement} - в начале

```bash
text="hello world"

echo "${text/#hello/hi}"   # hi world
echo "${text/#world/hi}"   # hello world (не в начале)
```

### ${var/%pattern/replacement} - в конце

```bash
text="hello world"

echo "${text/%world/earth}"  # hello earth
echo "${text/%hello/hi}"     # hello world (не в конце)
```

## Изменение регистра (bash 4.0+)

```bash
text="Hello World"

# Всё в нижний регистр
echo "${text,,}"           # hello world

# Всё в верхний регистр
echo "${text^^}"           # HELLO WORLD

# Первый символ в нижний
echo "${text,}"            # hello World

# Первый символ в верхний
echo "${text^}"            # Hello World

# Отдельные символы
echo "${text,,[HW]}"       # hello world (только H и W в нижний)
```

## Косвенное раскрытие

```bash
var_name="greeting"
greeting="Hello, World!"

# ${!name} - использовать значение name как имя переменной
echo "${!var_name}"        # Hello, World!

# Список переменных с префиксом
USER_NAME="John"
USER_AGE="25"
USER_CITY="Moscow"

echo "${!USER_@}"          # USER_AGE USER_CITY USER_NAME
echo "${!USER_*}"          # USER_AGE USER_CITY USER_NAME
```

## Практические примеры

### Обработка файлов

```bash
#!/bin/bash

file="/path/to/image.jpg"

dir="${file%/*}"
name="${file##*/}"
base="${name%.*}"
ext="${name##*.}"

echo "Директория: $dir"    # /path/to
echo "Файл: $name"         # image.jpg
echo "Имя: $base"          # image
echo "Расширение: $ext"    # jpg

# Новое имя
new_file="${dir}/${base}_backup.${ext}"
echo "Новый файл: $new_file"  # /path/to/image_backup.jpg
```

### Очистка строки

```bash
#!/bin/bash

# Удаление ведущих и концевых пробелов
text="   Hello World   "

# Удаление ведущих пробелов
text="${text#"${text%%[![:space:]]*}"}"

# Удаление концевых пробелов
text="${text%"${text##*[![:space:]]}"}"

echo "[$text]"  # [Hello World]
```

### URL парсинг

```bash
#!/bin/bash

url="https://user:pass@example.com:8080/path?query=value"

# Протокол
protocol="${url%%://*}"      # https

# Без протокола
rest="${url#*://}"           # user:pass@example.com:8080/path?query=value

# Пользователь
userinfo="${rest%%@*}"       # user:pass
user="${userinfo%%:*}"       # user
pass="${userinfo#*:}"        # pass

# Хост и путь
hostpath="${rest#*@}"        # example.com:8080/path?query=value
host="${hostpath%%:*}"       # example.com
```

---

[prev: 02-for-c-style](../10-for-loop/02-for-c-style.md) | [next: 02-string-operations](./02-string-operations.md)
