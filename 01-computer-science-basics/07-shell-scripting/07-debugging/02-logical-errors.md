# Логические ошибки

## Что такое логические ошибки

Логические ошибки - это ошибки в логике программы. Скрипт выполняется без синтаксических ошибок, но даёт неправильный результат. Такие ошибки труднее обнаружить.

## Распространённые логические ошибки

### 1. Неправильные условия

```bash
# Ошибка: использование = вместо -eq для чисел
if [[ "$count" = "5" ]]; then  # Строковое сравнение!
    echo "Равно 5"
fi

# Проблема: "5" = "5", но " 5" != "5"
count=" 5"
if [[ "$count" = "5" ]]; then  # false
    echo "Равно 5"
fi

# ПРАВИЛЬНО
if [[ $count -eq 5 ]]; then  # Числовое сравнение
    echo "Равно 5"
fi
```

### 2. Off-by-one (ошибка на единицу)

```bash
# Ошибка: < вместо <=
for ((i=0; i<5; i++)); do
    echo $i
done
# Выводит: 0 1 2 3 4 (5 итераций)

# Если нужно 5 включительно:
for ((i=0; i<=5; i++)); do
    echo $i
done
# Выводит: 0 1 2 3 4 5 (6 итераций)
```

### 3. Неинициализированные переменные

```bash
# Ошибка: typo в имени переменной
counter=0
((conter++))  # Опечатка! Создаётся новая переменная
echo $counter  # 0

# Защита: set -u
set -u
((conter++))  # bash: conter: unbound variable
```

### 4. Проблемы с кавычками

```bash
# Ошибка: пропущенные кавычки
filename="my file.txt"

if [[ -f $filename ]]; then  # В [[ ]] работает
    cat $filename  # ОШИБКА! Разбивается на "my" и "file.txt"
fi

# ПРАВИЛЬНО
if [[ -f "$filename" ]]; then
    cat "$filename"
fi
```

### 5. Неправильная логика условий

```bash
# Ошибка: && вместо ||
if [[ -z "$name" && -z "$email" ]]; then
    echo "Укажите имя И email"  # Оба должны быть пусты
fi

# Если нужно "хотя бы одно пусто":
if [[ -z "$name" || -z "$email" ]]; then
    echo "Укажите имя или email"
fi

# Ошибка: забыли !
if [[ -f "$file" ]]; then
    echo "Файл НЕ существует"  # Логика перепутана!
fi

# ПРАВИЛЬНО
if [[ ! -f "$file" ]]; then
    echo "Файл не существует"
fi
```

### 6. Проблемы с областью видимости

```bash
# Ошибка: переменная в pipe
count=0
cat file.txt | while read line; do
    ((count++))
done
echo "Строк: $count"  # 0! (subshell)

# ПРАВИЛЬНО
count=0
while read line; do
    ((count++))
done < file.txt
echo "Строк: $count"
```

### 7. Неправильный порядок операций

```bash
# Ошибка: удаление до проверки
rm "$file"
if [[ $? -ne 0 ]]; then
    echo "Не удалось удалить файл"
    # Но файл уже пытались удалить!
fi

# ПРАВИЛЬНО
if rm "$file"; then
    echo "Файл удалён"
else
    echo "Не удалось удалить файл"
fi
```

### 8. Бесконечные циклы

```bash
# Ошибка: условие никогда не станет ложным
while [[ $i -lt 10 ]]; do
    echo $i
    # ((i++))  # Забыли инкремент!
done

# Ошибка: изменение не той переменной
i=0
while [[ $i -lt 10 ]]; do
    echo $i
    ((j++))  # Опечатка
done
```

### 9. Проблемы с возвратом из функции

```bash
# Ошибка: echo вместо return
is_valid() {
    if [[ "$1" =~ ^[0-9]+$ ]]; then
        echo "true"
    else
        echo "false"
    fi
}

# Неправильное использование
if is_valid "123"; then  # Всегда true, т.к. вывод не пуст
    echo "Valid"
fi

# ПРАВИЛЬНО
is_valid() {
    [[ "$1" =~ ^[0-9]+$ ]]
}

if is_valid "123"; then
    echo "Valid"
fi
```

### 10. Race conditions

```bash
# Ошибка: проверка и действие не атомарны
if [[ ! -f "$lockfile" ]]; then
    touch "$lockfile"
    # Между проверкой и созданием другой процесс мог создать файл!
fi

# ПРАВИЛЬНО: атомарная операция
if mkdir "$lockdir" 2>/dev/null; then
    # Получили блокировку
    :
fi
```

## Обнаружение логических ошибок

### Вывод отладочной информации

```bash
#!/bin/bash

debug=true

debug_log() {
    if $debug; then
        echo "[DEBUG] $*" >&2
    fi
}

process() {
    local input=$1
    debug_log "Вход в process с input='$input'"

    # ... логика ...

    debug_log "Результат: $result"
}
```

### Проверка промежуточных значений

```bash
#!/bin/bash

calculate() {
    local a=$1
    local b=$2

    echo "a=$a, b=$b" >&2  # Отладка

    local sum=$((a + b))
    echo "sum=$sum" >&2  # Отладка

    local result=$((sum * 2))
    echo "result=$result" >&2  # Отладка

    echo $result
}
```

### Assert-функция

```bash
assert() {
    local condition=$1
    local message=${2:-"Assertion failed"}

    if ! eval "$condition"; then
        echo "ASSERT FAILED: $message" >&2
        echo "Condition: $condition" >&2
        exit 1
    fi
}

# Использование
count=5
assert '[[ $count -gt 0 ]]' "count должен быть положительным"
assert '[[ $count -lt 100 ]]' "count должен быть меньше 100"
```

## Защитное программирование

### set -e (exit on error)

```bash
#!/bin/bash
set -e  # Выход при первой ошибке

cd /nonexistent  # Скрипт остановится здесь
echo "Этого не будет"
```

### set -u (unbound variable)

```bash
#!/bin/bash
set -u  # Ошибка при использовании неопределённой переменной

echo $undefined_var  # Ошибка!
```

### set -o pipefail

```bash
#!/bin/bash
set -o pipefail  # Ошибка в pipe приводит к ошибке всей команды

false | true  # Вернёт 1 (ошибка от false)
```

### Комбинация

```bash
#!/bin/bash
set -euo pipefail

# Теперь скрипт более устойчив к ошибкам
```

## Тестирование логики

### Граничные случаи

```bash
# Тестируйте:
# - Пустой ввод
# - Нулевые значения
# - Отрицательные числа
# - Очень большие значения
# - Специальные символы
# - Пробелы в именах файлов

test_function() {
    # Пустой ввод
    result=$(my_function "")
    [[ "$result" == "expected" ]] || echo "FAIL: empty input"

    # Пробелы
    result=$(my_function "hello world")
    [[ "$result" == "expected" ]] || echo "FAIL: spaces"

    # Специальные символы
    result=$(my_function "test\$var")
    [[ "$result" == "expected" ]] || echo "FAIL: special chars"
}
```

### Пошаговое выполнение

```bash
# Выполняйте скрипт построчно в голове или на бумаге
# Отслеживайте значения переменных на каждом шаге

# Пример таблицы трассировки:
# Строка | i | count | result
# 5      | 0 | 0     | -
# 6      | 0 | 1     | -
# 7      | 1 | 1     | -
# ...
```
