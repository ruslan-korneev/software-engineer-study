# Оператор if

[prev: 03-script-structure](../03-top-down-design/03-script-structure.md) | [next: 02-exit-status](./02-exit-status.md)

---
## Базовый синтаксис

Оператор `if` позволяет выполнять код в зависимости от условия:

```bash
if условие; then
    # код, выполняемый если условие истинно
fi
```

### Простой пример

```bash
#!/bin/bash

if [[ -f "/etc/passwd" ]]; then
    echo "Файл /etc/passwd существует"
fi
```

## Конструкция if-else

```bash
if условие; then
    # код при истинном условии
else
    # код при ложном условии
fi
```

### Пример

```bash
#!/bin/bash

file="/tmp/test.txt"

if [[ -f "$file" ]]; then
    echo "Файл существует"
else
    echo "Файл не найден"
fi
```

## Конструкция if-elif-else

Для проверки нескольких условий:

```bash
if условие1; then
    # код при условии1
elif условие2; then
    # код при условии2
elif условие3; then
    # код при условии3
else
    # код, если ни одно условие не истинно
fi
```

### Пример с elif

```bash
#!/bin/bash

score=75

if [[ $score -ge 90 ]]; then
    echo "Оценка: Отлично"
elif [[ $score -ge 80 ]]; then
    echo "Оценка: Хорошо"
elif [[ $score -ge 70 ]]; then
    echo "Оценка: Удовлетворительно"
elif [[ $score -ge 60 ]]; then
    echo "Оценка: Посредственно"
else
    echo "Оценка: Неудовлетворительно"
fi
```

## Форматирование

### Несколько вариантов записи

```bash
# Вариант 1: then на той же строке
if [[ условие ]]; then
    команда
fi

# Вариант 2: then на новой строке
if [[ условие ]]
then
    команда
fi

# Вариант 3: однострочный (для простых случаев)
if [[ условие ]]; then команда; fi

# Вариант 4: с использованием && (альтернатива для простых случаев)
[[ условие ]] && команда
```

## Вложенные if

```bash
#!/bin/bash

file="/tmp/test.txt"

if [[ -e "$file" ]]; then
    echo "Объект существует"

    if [[ -f "$file" ]]; then
        echo "Это файл"

        if [[ -r "$file" ]]; then
            echo "Файл доступен для чтения"
        else
            echo "Нет прав на чтение"
        fi
    elif [[ -d "$file" ]]; then
        echo "Это директория"
    fi
else
    echo "Объект не существует"
fi
```

**Совет:** Избегайте глубокой вложенности - используйте ранний выход или функции.

## Альтернативы с && и ||

### Использование &&

Выполнить команду, если условие истинно:

```bash
# Вместо
if [[ -f "$file" ]]; then
    cat "$file"
fi

# Можно написать
[[ -f "$file" ]] && cat "$file"
```

### Использование ||

Выполнить команду, если условие ложно:

```bash
# Вместо
if [[ ! -d "$dir" ]]; then
    mkdir "$dir"
fi

# Можно написать
[[ -d "$dir" ]] || mkdir "$dir"
```

### Комбинация && и ||

```bash
# Тернарный оператор (условие ? да : нет)
[[ $age -ge 18 ]] && echo "Взрослый" || echo "Несовершеннолетний"
```

**Осторожно:** Если команда после `&&` завершится с ошибкой, выполнится команда после `||`.

## Практические примеры

### Проверка аргументов

```bash
#!/bin/bash

if [[ $# -eq 0 ]]; then
    echo "Использование: $0 <имя_файла>"
    exit 1
fi

filename=$1

if [[ -f "$filename" ]]; then
    echo "Обработка файла: $filename"
    cat "$filename"
else
    echo "Ошибка: файл '$filename' не найден"
    exit 1
fi
```

### Проверка пользователя

```bash
#!/bin/bash

if [[ $EUID -eq 0 ]]; then
    echo "Скрипт запущен от root"
else
    echo "Скрипт запущен от обычного пользователя"
    echo "Некоторые операции могут потребовать sudo"
fi
```

### Проверка ОС

```bash
#!/bin/bash

if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    echo "Linux"
    PACKAGE_MANAGER="apt"
elif [[ "$OSTYPE" == "darwin"* ]]; then
    echo "macOS"
    PACKAGE_MANAGER="brew"
elif [[ "$OSTYPE" == "cygwin" ]] || [[ "$OSTYPE" == "msys" ]]; then
    echo "Windows (Cygwin/MSYS)"
else
    echo "Неизвестная ОС: $OSTYPE"
fi
```

### Валидация ввода

```bash
#!/bin/bash

read -p "Введите число от 1 до 10: " number

if [[ ! "$number" =~ ^[0-9]+$ ]]; then
    echo "Ошибка: введите число"
    exit 1
fi

if [[ $number -lt 1 ]] || [[ $number -gt 10 ]]; then
    echo "Ошибка: число должно быть от 1 до 10"
    exit 1
fi

echo "Вы ввели: $number"
```

### Проверка доступности сервиса

```bash
#!/bin/bash

host="google.com"
port=80

if nc -z -w5 "$host" "$port" 2>/dev/null; then
    echo "Сервис $host:$port доступен"
else
    echo "Сервис $host:$port недоступен"
fi
```

## Команда true и false

`true` и `false` - встроенные команды, возвращающие код 0 и 1 соответственно:

```bash
if true; then
    echo "Это всегда выполнится"
fi

if false; then
    echo "Это никогда не выполнится"
else
    echo "А это выполнится"
fi
```

## Пустая команда :

Команда `:` (двоеточие) - это null command, всегда возвращает true:

```bash
if [[ условие ]]; then
    :  # Ничего не делаем (placeholder)
else
    echo "Условие ложно"
fi
```

## Распространённые ошибки

### Ошибка: пробелы в присваивании

```bash
# НЕПРАВИЛЬНО
if [ $var=value ]; then  # Это не сравнение!

# ПРАВИЛЬНО
if [ "$var" = "value" ]; then
```

### Ошибка: пропущенные кавычки

```bash
var="hello world"

# НЕПРАВИЛЬНО - разбивается на несколько аргументов
if [ $var = "hello world" ]; then

# ПРАВИЛЬНО
if [ "$var" = "hello world" ]; then
```

### Ошибка: использование = вместо ==

```bash
# В [ ] рекомендуется =
if [ "$var" = "value" ]; then

# В [[ ]] можно использовать и =, и ==
if [[ "$var" == "value" ]]; then
```

---

[prev: 03-script-structure](../03-top-down-design/03-script-structure.md) | [next: 02-exit-status](./02-exit-status.md)
