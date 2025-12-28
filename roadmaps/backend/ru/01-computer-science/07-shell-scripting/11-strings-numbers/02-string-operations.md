# Операции со строками

[prev: 01-parameter-expansion](./01-parameter-expansion.md) | [next: 03-arithmetic](./03-arithmetic.md)

---
## Конкатенация

```bash
#!/bin/bash

first="Hello"
second="World"

# Простая конкатенация
result="$first $second"
echo "$result"  # Hello World

# Конкатенация с литералами
result="${first}, ${second}!"
echo "$result"  # Hello, World!

# Добавление к переменной
str="Hello"
str+=" World"
echo "$str"  # Hello World

# Множественная конкатенация
full_name="${first_name}${middle_name:+ $middle_name} ${last_name}"
```

## Длина строки

```bash
#!/bin/bash

text="Hello, World!"

# Через parameter expansion
echo "${#text}"  # 13

# Через wc
echo -n "$text" | wc -c  # 13

# Через expr (устаревший способ)
expr length "$text"  # 13
```

## Извлечение подстроки

```bash
#!/bin/bash

text="Hello, World!"

# ${var:offset:length}
echo "${text:0:5}"    # Hello
echo "${text:7:5}"    # World
echo "${text:7}"      # World!

# Отрицательный offset (от конца)
echo "${text: -6}"    # World!
echo "${text: -6:5}"  # World

# Через cut
echo "$text" | cut -c1-5   # Hello
echo "$text" | cut -c8-12  # World
```

## Поиск в строке

### Проверка содержания

```bash
#!/bin/bash

text="Hello, World!"

# Через [[ ]]
if [[ "$text" == *"World"* ]]; then
    echo "Содержит 'World'"
fi

# Regex
if [[ "$text" =~ World ]]; then
    echo "Содержит 'World'"
fi

# Через grep
if echo "$text" | grep -q "World"; then
    echo "Содержит 'World'"
fi
```

### Позиция подстроки

```bash
#!/bin/bash

text="Hello, World!"

# Через awk
pos=$(echo "$text" | awk '{print index($0, "World")}')
echo "Позиция: $pos"  # 8

# Через expr
pos=$(expr index "$text" "W")
echo "Позиция W: $pos"  # 8
```

## Замена в строках

```bash
#!/bin/bash

text="hello hello hello"

# Первое вхождение
echo "${text/hello/hi}"      # hi hello hello

# Все вхождения
echo "${text//hello/hi}"     # hi hi hi

# В начале строки
echo "${text/#hello/hi}"     # hi hello hello

# В конце строки
text="hello world"
echo "${text/%world/earth}"  # hello earth

# Удаление (замена на пустую строку)
echo "${text//o/}"           # hell wrld

# Через sed
echo "$text" | sed 's/hello/hi/g'
```

## Разделение строки

```bash
#!/bin/bash

# Через IFS и read
text="apple,banana,cherry"
IFS=',' read -ra fruits <<< "$text"
echo "${fruits[0]}"  # apple
echo "${fruits[1]}"  # banana

# Через cut
echo "$text" | cut -d',' -f1  # apple
echo "$text" | cut -d',' -f2  # banana

# Через awk
echo "$text" | awk -F',' '{print $1}'  # apple

# Разделение PATH
IFS=':' read -ra paths <<< "$PATH"
for p in "${paths[@]}"; do
    echo "$p"
done
```

## Преобразование регистра

```bash
#!/bin/bash

text="Hello World"

# Bash 4.0+
echo "${text,,}"           # hello world (всё в нижний)
echo "${text^^}"           # HELLO WORLD (всё в верхний)
echo "${text,}"            # hello World (первый в нижний)
echo "${text^}"            # Hello World (первый в верхний)

# Через tr
echo "$text" | tr '[:upper:]' '[:lower:]'  # hello world
echo "$text" | tr '[:lower:]' '[:upper:]'  # HELLO WORLD

# Через awk
echo "$text" | awk '{print tolower($0)}'
echo "$text" | awk '{print toupper($0)}'
```

## Удаление пробелов

```bash
#!/bin/bash

text="   Hello World   "

# Удаление ведущих пробелов
trimmed="${text#"${text%%[![:space:]]*}"}"
echo "[$trimmed]"  # [Hello World   ]

# Удаление концевых пробелов
trimmed="${text%"${text##*[![:space:]]}"}"
echo "[$trimmed]"  # [   Hello World]

# Через xargs (удаляет и ведущие, и концевые)
trimmed=$(echo "$text" | xargs)
echo "[$trimmed]"  # [Hello World]

# Через sed
echo "$text" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'
```

## Проверка строк

```bash
#!/bin/bash

# Пустая строка
str=""
[[ -z "$str" ]] && echo "Пустая"

# Непустая строка
str="hello"
[[ -n "$str" ]] && echo "Непустая"

# Начинается с
str="Hello, World"
[[ "$str" == Hello* ]] && echo "Начинается с 'Hello'"

# Заканчивается на
[[ "$str" == *World ]] && echo "Заканчивается на 'World'"

# Соответствует паттерну
[[ "$str" == *", "* ]] && echo "Содержит запятую с пробелом"

# Regex
[[ "$str" =~ ^[A-Z] ]] && echo "Начинается с заглавной"
```

## Практические примеры

### Валидация email

```bash
#!/bin/bash

is_valid_email() {
    local email=$1
    local pattern='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

    [[ "$email" =~ $pattern ]]
}

if is_valid_email "user@example.com"; then
    echo "Email валиден"
fi
```

### Извлечение доменного имени

```bash
#!/bin/bash

url="https://www.example.com/path/to/page"

# Удаляем протокол
no_protocol="${url#*://}"

# Извлекаем домен
domain="${no_protocol%%/*}"

echo "Домен: $domain"  # www.example.com
```

### Генерация slug из заголовка

```bash
#!/bin/bash

create_slug() {
    local title=$1

    # В нижний регистр
    local slug="${title,,}"

    # Заменяем пробелы на дефисы
    slug="${slug// /-}"

    # Удаляем всё кроме букв, цифр и дефисов
    slug=$(echo "$slug" | tr -cd 'a-z0-9-')

    # Удаляем множественные дефисы
    slug=$(echo "$slug" | tr -s '-')

    echo "$slug"
}

slug=$(create_slug "Hello World! This is a Test")
echo "$slug"  # hello-world-this-is-a-test
```

### Маскирование пароля

```bash
#!/bin/bash

mask_password() {
    local password=$1
    local length=${#password}

    if [[ $length -le 4 ]]; then
        echo "****"
    else
        # Показать первые 2 и последние 2 символа
        local start="${password:0:2}"
        local end="${password: -2}"
        local middle=""
        for ((i = 0; i < length - 4; i++)); do
            middle+="*"
        done
        echo "${start}${middle}${end}"
    fi
}

mask_password "secret123"  # se*****23
```

---

[prev: 01-parameter-expansion](./01-parameter-expansion.md) | [next: 03-arithmetic](./03-arithmetic.md)
