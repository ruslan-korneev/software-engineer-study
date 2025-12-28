# Переменная IFS

[prev: 01-read-command](./01-read-command.md) | [next: 03-validating-input](./03-validating-input.md)

---
## Что такое IFS

**IFS** (Internal Field Separator) - это специальная переменная, определяющая символы-разделители для разбиения строк на слова. По умолчанию IFS содержит пробел, табуляцию и новую строку.

```bash
# Посмотреть текущее значение IFS
echo "$IFS" | cat -A
# Вывод:  ^I$  (пробел, табуляция, новая строка)

# Или через printf
printf '%q\n' "$IFS"
# $' \t\n'
```

## Как работает IFS

### Влияние на разбиение слов

```bash
line="apple  banana   cherry"

# С IFS по умолчанию (пробелы)
read -ra words <<< "$line"
echo "Количество слов: ${#words[@]}"  # 3
echo "Слова: ${words[@]}"  # apple banana cherry

# Множественные пробелы сжимаются
```

### Изменение IFS

```bash
# Изменить разделитель на двоеточие
IFS=':'
line="apple:banana:cherry"
read -ra words <<< "$line"
echo "${words[@]}"  # apple banana cherry

# Вернуть значение по умолчанию
IFS=$' \t\n'
```

## IFS и команда read

### Разбиение по разделителю

```bash
# Разбиение пути PATH
IFS=':'
read -ra paths <<< "$PATH"
for path in "${paths[@]}"; do
    echo "$path"
done
```

### Чтение CSV

```bash
# Файл data.csv:
# name,age,city
# Alice,25,Moscow

while IFS=',' read -r name age city; do
    echo "Имя: $name, Возраст: $age, Город: $city"
done < data.csv
```

### Локальное изменение IFS

```bash
# Способ 1: Присвоение перед командой (только для этой команды)
while IFS=':' read -r field1 field2 rest; do
    echo "$field1 - $field2"
done < /etc/passwd

# Способ 2: Сохранение и восстановление
OLD_IFS="$IFS"
IFS=':'
# ... работа с новым IFS ...
IFS="$OLD_IFS"

# Способ 3: Подоболочка
(
    IFS=':'
    # ... работа с новым IFS ...
)
# Вне подоболочки IFS не изменился
```

## Важные особенности IFS

### Пустой IFS

Если IFS пуст, разбиение не происходит:

```bash
IFS=
line="one two three"
read -ra words <<< "$line"
echo "Слов: ${#words[@]}"  # 1 (вся строка как одно слово)
echo "${words[0]}"  # one two three
```

### IFS= для сохранения пробелов

```bash
# Без IFS= ведущие/концевые пробелы удаляются
line="  hello world  "
read -r var <<< "$line"
echo "[$var]"  # [hello world]

# С IFS= пробелы сохраняются
IFS= read -r var <<< "$line"
echo "[$var]"  # [  hello world  ]
```

### Несколько разделителей

```bash
# Несколько символов в IFS
IFS=':,'
line="apple:banana,cherry:date"
read -ra words <<< "$line"
echo "${words[@]}"  # apple banana cherry date
```

### Разделитель в начале/конце

```bash
IFS=':'
line=":one:two:"
read -ra words <<< "$line"
echo "Количество: ${#words[@]}"  # 3
echo "[${words[0]}]"  # [] (пустой элемент)
echo "[${words[1]}]"  # [one]
echo "[${words[2]}]"  # [two]
# Заметьте: конечный : не создаёт пустой элемент в read
```

## IFS и циклы

### Разбиение в for

```bash
# По умолчанию разбивает по пробелам
for word in $line; do
    echo "$word"
done

# С изменённым IFS
IFS=':'
for part in $PATH; do
    echo "$part"
done
```

### Обход строк файла

```bash
# Правильный способ чтения файла построчно
while IFS= read -r line; do
    echo "Строка: $line"
done < file.txt
```

## Практические примеры

### Парсинг /etc/passwd

```bash
#!/bin/bash

echo "Пользователи системы:"
while IFS=':' read -r user pass uid gid gecos home shell; do
    if [[ $uid -ge 1000 ]] && [[ "$shell" != "/usr/sbin/nologin" ]]; then
        echo "  $user (UID: $uid) - $home"
    fi
done < /etc/passwd
```

### Разбиение PATH

```bash
#!/bin/bash

echo "Директории в PATH:"
IFS=':' read -ra paths <<< "$PATH"
for i in "${!paths[@]}"; do
    path="${paths[$i]}"
    if [[ -d "$path" ]]; then
        echo "  $((i+1)). $path (существует)"
    else
        echo "  $((i+1)). $path (НЕ СУЩЕСТВУЕТ)"
    fi
done
```

### Парсинг ключ=значение

```bash
#!/bin/bash

config="
host=localhost
port=8080
debug=true
"

while IFS='=' read -r key value; do
    [[ -z "$key" ]] && continue  # Пропуск пустых строк
    [[ "$key" == \#* ]] && continue  # Пропуск комментариев
    echo "Ключ: $key, Значение: $value"
done <<< "$config"
```

### Обработка CSV с кавычками

```bash
#!/bin/bash

# Простой CSV парсер (без кавычек внутри полей)
parse_csv_line() {
    local line=$1
    local IFS=','
    local -a fields

    read -ra fields <<< "$line"
    printf '%s\n' "${fields[@]}"
}

csv_line='Alice,25,"New York"'
parse_csv_line "$csv_line"
```

### Безопасное разбиение с сохранением IFS

```bash
#!/bin/bash

split_string() {
    local string=$1
    local delimiter=${2:-:}
    local -n result_array=$3  # nameref (bash 4.3+)

    local IFS="$delimiter"
    read -ra result_array <<< "$string"
}

# Использование
my_string="one:two:three"
declare -a my_array
split_string "$my_string" ":" my_array

echo "Элементов: ${#my_array[@]}"
echo "Элементы: ${my_array[@]}"
```

## Best Practices

### 1. Всегда локализуйте изменения IFS

```bash
# Плохо - глобальное изменение
IFS=':'
# ... много кода ...
# где-то ошибка из-за изменённого IFS

# Хорошо - локальное изменение
parse_path() {
    local IFS=':'
    local -a paths
    read -ra paths <<< "$PATH"
    printf '%s\n' "${paths[@]}"
}
```

### 2. Используйте IFS= read -r для построчного чтения

```bash
while IFS= read -r line; do
    process "$line"
done < file.txt
```

### 3. Помните о значении по умолчанию

```bash
# Сброс к значению по умолчанию
IFS=$' \t\n'

# Или
unset IFS  # Также возвращает поведение по умолчанию
```

### 4. Используйте присвоение перед командой

```bash
# IFS действует только для этой команды
while IFS=':' read -r a b c; do
    echo "$a $b $c"
done < file.txt
# После цикла IFS не изменён
```

---

[prev: 01-read-command](./01-read-command.md) | [next: 03-validating-input](./03-validating-input.md)
