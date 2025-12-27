# Переменные в Bash

## Основы переменных

Переменные в bash - это именованные области памяти для хранения данных. В отличие от многих языков программирования, bash не требует объявления типа переменной.

### Создание и присваивание

```bash
# Присваивание (БЕЗ пробелов вокруг =)
name="John"
age=25
path=/home/user

# НЕПРАВИЛЬНО - пробелы недопустимы!
name = "John"   # Ошибка: bash интерпретирует как команду "name"
```

### Обращение к переменной

```bash
name="World"

# Использование $ для получения значения
echo $name          # World
echo "Hello, $name" # Hello, World

# Фигурные скобки для явного разделения
echo "${name}!"     # World!
echo "$name_file"   # Пусто (ищет переменную name_file)
echo "${name}_file" # World_file
```

## Типы переменных

### Строковые переменные (по умолчанию)

Все переменные в bash по умолчанию являются строками:

```bash
number=42
echo $number    # 42 (но это строка "42")

# Конкатенация строк
first="Hello"
second="World"
result="$first $second"  # Hello World
result="${first}${second}"  # HelloWorld
```

### Целочисленные переменные

Можно объявить переменную как целочисленную:

```bash
declare -i num=10
num=num+5       # Автоматическое вычисление
echo $num       # 15

# Без declare -i пришлось бы писать:
num=$((num + 5))
```

### Массивы

```bash
# Индексированный массив
fruits=("apple" "banana" "cherry")
echo ${fruits[0]}  # apple

# Ассоциативный массив (bash 4+)
declare -A user
user[name]="John"
user[age]=25
echo ${user[name]}  # John
```

## Области видимости

### Локальные переменные

По умолчанию переменные локальны для текущего shell:

```bash
my_var="Hello"
bash -c 'echo $my_var'  # Пусто - дочерний процесс не видит
```

### Экспортированные переменные (переменные окружения)

```bash
export MY_VAR="Hello"
bash -c 'echo $MY_VAR'  # Hello - дочерний процесс видит

# Или в одну строку
export DATABASE_URL="postgres://localhost/db"

# Экспорт существующей переменной
config_path="/etc/app"
export config_path
```

### Локальные переменные в функциях

```bash
my_function() {
    local local_var="Я локальная"
    global_var="Я глобальная"
    echo "$local_var"
}

my_function
echo "$local_var"   # Пусто
echo "$global_var"  # Я глобальная
```

## Специальные переменные

### Позиционные параметры

```bash
# $0 - имя скрипта
# $1, $2, ... - аргументы
# $# - количество аргументов
# $@ - все аргументы как отдельные слова
# $* - все аргументы как одна строка

echo "Скрипт: $0"
echo "Первый аргумент: $1"
echo "Всего аргументов: $#"
echo "Все аргументы: $@"
```

### Переменные статуса

```bash
# $? - код возврата последней команды
ls /nonexistent 2>/dev/null
echo $?  # 1 (ошибка)

ls /tmp
echo $?  # 0 (успех)

# $$ - PID текущего shell
echo "Мой PID: $$"

# $! - PID последнего фонового процесса
sleep 10 &
echo "PID фонового процесса: $!"
```

## Значения по умолчанию

Bash предоставляет операторы для работы с неопределёнными переменными:

```bash
# ${var:-default} - использовать default, если var не определена или пуста
name=${1:-"Anonymous"}
echo "Hello, $name"

# ${var:=default} - присвоить default, если var не определена или пуста
: ${config:="/etc/default.conf"}
echo "Config: $config"

# ${var:?message} - вывести ошибку, если var не определена
: ${REQUIRED_VAR:?"Переменная REQUIRED_VAR должна быть задана"}

# ${var:+value} - использовать value, если var определена
debug=${DEBUG:+"[DEBUG] "}
echo "${debug}Starting..."  # Пусто, если DEBUG не задана
```

### Разница между `:` и без `:`

```bash
var=""  # Пустая строка

# С : - проверяет и на пустоту, и на неопределённость
echo ${var:-default}   # default (var пуста)

# Без : - проверяет только на неопределённость
echo ${var-default}    # (пустая строка - var определена)
```

## Кавычки и переменные

### Двойные кавычки

Внутри двойных кавычек переменные раскрываются:

```bash
name="World"
echo "Hello, $name"        # Hello, World
echo "Home is $HOME"       # Home is /home/user
echo "Command: $(whoami)"  # Command: user
```

### Одинарные кавычки

Внутри одинарных кавычек всё воспринимается буквально:

```bash
echo 'Hello, $name'        # Hello, $name
echo 'No $(commands)'      # No $(commands)
```

### Смешанное использование

```bash
# Часть в одинарных, часть в двойных
echo 'The value of $HOME is: '"$HOME"
# The value of $HOME is: /home/user
```

## Удаление переменных

```bash
my_var="Hello"
echo $my_var    # Hello

unset my_var
echo $my_var    # (пусто)

# Проверка на существование
if [[ -v my_var ]]; then
    echo "Переменная существует"
else
    echo "Переменная не существует"
fi
```

## Практические примеры

### Конфигурация скрипта

```bash
#!/bin/bash

# Значения по умолчанию
LOG_LEVEL=${LOG_LEVEL:-"INFO"}
OUTPUT_DIR=${OUTPUT_DIR:-"./output"}
MAX_RETRIES=${MAX_RETRIES:-3}

echo "Log level: $LOG_LEVEL"
echo "Output directory: $OUTPUT_DIR"
echo "Max retries: $MAX_RETRIES"
```

### Формирование путей

```bash
#!/bin/bash

base_dir="/var/www"
app_name="myapp"
version="1.0.0"

# Формирование путей
app_dir="${base_dir}/${app_name}"
release_dir="${app_dir}/releases/${version}"
config_file="${app_dir}/config/settings.conf"

echo "App directory: $app_dir"
echo "Release directory: $release_dir"
echo "Config file: $config_file"
```

### Временные файлы

```bash
#!/bin/bash

# Создание уникального временного файла
tmp_file=$(mktemp)
# или с шаблоном
tmp_file=$(mktemp /tmp/myapp.XXXXXX)

echo "Temporary file: $tmp_file"

# Очистка при выходе
trap "rm -f $tmp_file" EXIT
```
