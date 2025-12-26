# Создание массивов

## Типы массивов в Bash

Bash поддерживает два типа массивов:
1. **Индексированные массивы** - с числовыми индексами (0, 1, 2, ...)
2. **Ассоциативные массивы** - с строковыми ключами (требуют bash 4.0+)

## Создание индексированных массивов

### Явная инициализация

```bash
#!/bin/bash

# Способ 1: Присваивание списка
fruits=("apple" "banana" "cherry")

# Способ 2: Присваивание отдельных элементов
colors[0]="red"
colors[1]="green"
colors[2]="blue"

# Способ 3: Через declare
declare -a numbers=(1 2 3 4 5)
```

### Элементы с пробелами

```bash
#!/bin/bash

# Кавычки сохраняют пробелы
names=("John Doe" "Jane Smith" "Bob Wilson")

echo "${names[0]}"  # John Doe
echo "${names[1]}"  # Jane Smith
```

### Несмежные индексы

```bash
#!/bin/bash

# Индексы не обязаны быть последовательными
sparse[0]="first"
sparse[5]="sixth"
sparse[100]="hundredth"

echo "${sparse[5]}"  # sixth
```

## Создание из других источников

### Из вывода команды

```bash
#!/bin/bash

# Разбиение по пробелам/переносам
files=($(ls))

# Безопаснее для файлов с пробелами:
mapfile -t files < <(ls)
# или
readarray -t files < <(ls)
```

### Из строки

```bash
#!/bin/bash

string="one two three"
words=($string)  # Разбиение по IFS

echo "${words[0]}"  # one
echo "${words[1]}"  # two
```

### Из файла

```bash
#!/bin/bash

# Построчное чтение в массив
mapfile -t lines < file.txt
# или
readarray -t lines < file.txt

# Старый способ
while IFS= read -r line; do
    lines+=("$line")
done < file.txt
```

### С разделителем

```bash
#!/bin/bash

path="/usr/local/bin"
IFS='/' read -ra parts <<< "$path"

# parts = ("" "usr" "local" "bin")
```

## Объявление с declare

```bash
#!/bin/bash

# Явное объявление индексированного массива
declare -a my_array

# Readonly массив
declare -ra readonly_array=("can't" "change" "this")

# Числовой массив
declare -ai int_array=(1 2 3)
int_array[0]=5       # OK
int_array[1]="abc"   # Станет 0 (не число)
```

## Создание ассоциативных массивов

### Объявление обязательно

```bash
#!/bin/bash

# Ассоциативные массивы ТРЕБУЮТ declare -A
declare -A user

user[name]="John"
user[age]=25
user[city]="Moscow"

echo "${user[name]}"  # John
echo "${user[age]}"   # 25
```

### Инициализация при создании

```bash
#!/bin/bash

declare -A config=(
    [host]="localhost"
    [port]="8080"
    [debug]="true"
)

echo "${config[host]}"  # localhost
```

## Копирование массивов

```bash
#!/bin/bash

original=("one" "two" "three")

# Копирование
copy=("${original[@]}")

# Изменение копии не влияет на оригинал
copy[0]="ONE"
echo "${original[0]}"  # one
echo "${copy[0]}"      # ONE
```

## Пустой массив

```bash
#!/bin/bash

# Создание пустого массива
empty=()
declare -a also_empty

# Проверка на пустоту
if [[ ${#empty[@]} -eq 0 ]]; then
    echo "Массив пуст"
fi
```

## Практические примеры

### Массив из аргументов скрипта

```bash
#!/bin/bash

# Все аргументы в массив
args=("$@")

echo "Количество аргументов: ${#args[@]}"
for arg in "${args[@]}"; do
    echo "  - $arg"
done
```

### Массив файлов

```bash
#!/bin/bash

# Все .txt файлы в массив
shopt -s nullglob  # Пустой массив если нет совпадений
txt_files=(*.txt)

if [[ ${#txt_files[@]} -eq 0 ]]; then
    echo "Нет .txt файлов"
else
    echo "Найдено ${#txt_files[@]} файлов:"
    printf '%s\n' "${txt_files[@]}"
fi
```

### Конфигурационный массив

```bash
#!/bin/bash

declare -A config

# Загрузка конфигурации
while IFS='=' read -r key value; do
    [[ -z "$key" || "$key" == \#* ]] && continue
    config[$key]=$value
done < config.ini

# Использование
echo "Host: ${config[host]}"
echo "Port: ${config[port]}"
```

### Многомерный массив (симуляция)

```bash
#!/bin/bash

# Bash не поддерживает многомерные массивы,
# но можно симулировать через ключи
declare -A matrix

# Заполнение
for ((i=0; i<3; i++)); do
    for ((j=0; j<3; j++)); do
        matrix[$i,$j]=$((i + j))
    done
done

# Доступ
echo "matrix[1,2] = ${matrix[1,2]}"  # 3

# Вывод матрицы
for ((i=0; i<3; i++)); do
    for ((j=0; j<3; j++)); do
        printf "%3d" "${matrix[$i,$j]}"
    done
    echo
done
```

## Ограничения массивов

```bash
#!/bin/bash

# Нет вложенных массивов
# nested=((1 2) (3 4))  # Ошибка!

# Нельзя экспортировать массивы в окружение
declare -a arr=(1 2 3)
export arr  # Не сработает как ожидается

# Ключи ассоциативных массивов должны быть строками
declare -A hash
hash[123]="value"     # 123 становится строкой "123"
```
