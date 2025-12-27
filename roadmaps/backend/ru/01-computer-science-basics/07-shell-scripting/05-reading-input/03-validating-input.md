# Валидация ввода

## Зачем валидировать ввод

Пользовательский ввод нельзя считать надёжным. Валидация защищает от:
- Некорректных данных
- Ошибок выполнения
- Проблем безопасности
- Неожиданного поведения скрипта

## Проверка на пустоту

```bash
read -rp "Введите имя: " name

# Способ 1: через -z
if [[ -z "$name" ]]; then
    echo "Ошибка: имя не может быть пустым"
    exit 1
fi

# Способ 2: через -n (не пусто)
if [[ -n "$name" ]]; then
    echo "Привет, $name!"
fi

# Способ 3: значение по умолчанию
name="${name:-Anonymous}"
```

### Цикл до получения непустого ввода

```bash
while true; do
    read -rp "Введите имя: " name
    if [[ -n "$name" ]]; then
        break
    fi
    echo "Имя не может быть пустым!"
done
```

## Проверка чисел

### Целые числа

```bash
is_integer() {
    [[ "$1" =~ ^-?[0-9]+$ ]]
}

read -rp "Введите число: " num

if is_integer "$num"; then
    echo "Это целое число: $num"
else
    echo "Это не целое число!"
fi
```

### Положительные целые

```bash
is_positive_integer() {
    [[ "$1" =~ ^[0-9]+$ ]] && [[ "$1" -gt 0 ]]
}

if is_positive_integer "$num"; then
    echo "Положительное число"
fi
```

### Числа с плавающей точкой

```bash
is_float() {
    [[ "$1" =~ ^-?[0-9]*\.?[0-9]+$ ]]
}

# Примеры: 3.14, -2.5, .5, 10
```

### Проверка диапазона

```bash
in_range() {
    local value=$1
    local min=$2
    local max=$3

    is_integer "$value" && [[ $value -ge $min && $value -le $max ]]
}

read -rp "Введите возраст (1-120): " age

if in_range "$age" 1 120; then
    echo "Возраст: $age"
else
    echo "Некорректный возраст"
fi
```

## Проверка строк

### Допустимые символы

```bash
# Только буквы
is_alpha() {
    [[ "$1" =~ ^[a-zA-Z]+$ ]]
}

# Только буквы и цифры
is_alnum() {
    [[ "$1" =~ ^[a-zA-Z0-9]+$ ]]
}

# Допустимое имя переменной
is_valid_varname() {
    [[ "$1" =~ ^[a-zA-Z_][a-zA-Z0-9_]*$ ]]
}
```

### Проверка длины

```bash
check_length() {
    local str=$1
    local min=${2:-0}
    local max=${3:-999999}
    local len=${#str}

    [[ $len -ge $min && $len -le $max ]]
}

read -rp "Пароль (8-20 символов): " password

if check_length "$password" 8 20; then
    echo "Пароль принят"
else
    echo "Пароль должен быть от 8 до 20 символов"
fi
```

### Проверка формата

```bash
# Email (упрощённая проверка)
is_email() {
    [[ "$1" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

# Дата в формате YYYY-MM-DD
is_date() {
    [[ "$1" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]
}

# IP адрес (IPv4)
is_ipv4() {
    local ip=$1
    local IFS='.'
    local -a octets
    read -ra octets <<< "$ip"

    [[ ${#octets[@]} -eq 4 ]] || return 1

    for octet in "${octets[@]}"; do
        [[ "$octet" =~ ^[0-9]+$ ]] || return 1
        [[ $octet -ge 0 && $octet -le 255 ]] || return 1
    done
    return 0
}

# URL (базовая проверка)
is_url() {
    [[ "$1" =~ ^https?://[a-zA-Z0-9.-]+(/.*)?$ ]]
}
```

## Проверка файлов и путей

```bash
validate_file() {
    local file=$1

    if [[ -z "$file" ]]; then
        echo "Файл не указан" >&2
        return 1
    fi

    if [[ ! -e "$file" ]]; then
        echo "Файл не существует: $file" >&2
        return 1
    fi

    if [[ ! -f "$file" ]]; then
        echo "Это не файл: $file" >&2
        return 1
    fi

    if [[ ! -r "$file" ]]; then
        echo "Нет прав на чтение: $file" >&2
        return 1
    fi

    return 0
}

read -rp "Путь к файлу: " filepath

if validate_file "$filepath"; then
    cat "$filepath"
fi
```

### Безопасная проверка пути

```bash
# Проверка на path traversal
is_safe_path() {
    local path=$1
    local base_dir=$2

    # Получаем абсолютный путь
    local real_path
    real_path=$(realpath -m "$path" 2>/dev/null) || return 1

    # Проверяем, что путь внутри разрешённой директории
    [[ "$real_path" == "$base_dir"/* ]]
}

# Использование
if is_safe_path "$user_input" "/var/data"; then
    cat "$user_input"
else
    echo "Недопустимый путь"
fi
```

## Выбор из списка

### Проверка допустимых значений

```bash
is_valid_option() {
    local value=$1
    shift
    local options=("$@")

    for opt in "${options[@]}"; do
        [[ "$value" == "$opt" ]] && return 0
    done
    return 1
}

valid_colors=("red" "green" "blue" "yellow")

read -rp "Выберите цвет (red/green/blue/yellow): " color

if is_valid_option "$color" "${valid_colors[@]}"; then
    echo "Вы выбрали: $color"
else
    echo "Недопустимый цвет"
fi
```

### Использование case

```bash
read -rp "Продолжить? (y/n): " answer

case "$answer" in
    [yY]|[yY][eE][sS]|[дД][аА])
        echo "Продолжаем..."
        ;;
    [nN]|[nN][oO]|[нН][еЕ][тТ])
        echo "Отмена"
        exit 0
        ;;
    *)
        echo "Некорректный ответ"
        exit 1
        ;;
esac
```

## Универсальная функция валидации

```bash
#!/bin/bash

# Универсальный валидатор
validate() {
    local value=$1
    local type=$2
    local extra=$3

    case "$type" in
        nonempty)
            [[ -n "$value" ]]
            ;;
        integer)
            [[ "$value" =~ ^-?[0-9]+$ ]]
            ;;
        positive)
            [[ "$value" =~ ^[0-9]+$ ]] && [[ "$value" -gt 0 ]]
            ;;
        range)
            local min max
            IFS='-' read -r min max <<< "$extra"
            [[ "$value" =~ ^-?[0-9]+$ ]] && \
            [[ "$value" -ge "$min" ]] && \
            [[ "$value" -le "$max" ]]
            ;;
        email)
            [[ "$value" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
            ;;
        file)
            [[ -f "$value" && -r "$value" ]]
            ;;
        dir)
            [[ -d "$value" ]]
            ;;
        *)
            echo "Unknown validation type: $type" >&2
            return 2
            ;;
    esac
}

# Использование
validate "hello@example.com" "email" && echo "Valid email"
validate "42" "range" "1-100" && echo "In range"
validate "" "nonempty" || echo "Empty value"
```

## Интерактивный ввод с валидацией

```bash
# Повторный запрос до получения валидного ввода
read_validated() {
    local prompt=$1
    local validator=$2
    local error_msg=${3:-"Некорректный ввод"}
    local value

    while true; do
        read -rp "$prompt" value
        if $validator "$value"; then
            echo "$value"
            return 0
        fi
        echo "$error_msg" >&2
    done
}

# Валидаторы
is_valid_age() { [[ "$1" =~ ^[0-9]+$ ]] && [[ "$1" -ge 1 && "$1" -le 120 ]]; }
is_valid_email() { [[ "$1" =~ ^[^@]+@[^@]+\.[^@]+$ ]]; }

# Использование
age=$(read_validated "Возраст: " is_valid_age "Введите число от 1 до 120")
email=$(read_validated "Email: " is_valid_email "Введите корректный email")

echo "Возраст: $age"
echo "Email: $email"
```

## Санитизация ввода

### Удаление опасных символов

```bash
sanitize_filename() {
    local name=$1
    # Удаляем всё кроме букв, цифр, точки, дефиса, подчёркивания
    echo "${name//[^a-zA-Z0-9._-]/}"
}

sanitize_for_sql() {
    local value=$1
    # Экранируем одинарные кавычки
    echo "${value//\'/\'\'}"
}
```

### Удаление управляющих символов

```bash
sanitize_input() {
    local input=$1
    # Удаляем непечатаемые символы
    echo "$input" | tr -cd '[:print:]'
}
```

## Best Practices

1. **Валидируйте ВСЁ** - никогда не доверяйте вводу
2. **Сообщайте об ошибках** понятным языком
3. **Используйте белый список** вместо чёрного
4. **Санитизируйте** перед использованием в командах
5. **Ограничивайте длину** для предотвращения переполнения
6. **Используйте функции** для переиспользования логики
