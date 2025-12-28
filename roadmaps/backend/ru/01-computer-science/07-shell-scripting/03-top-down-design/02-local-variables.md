# Локальные переменные

[prev: 01-functions](./01-functions.md) | [next: 03-script-structure](./03-script-structure.md)

---
## Проблема глобальных переменных

По умолчанию все переменные в bash являются глобальными:

```bash
count=0

increment() {
    count=$((count + 1))
}

other_function() {
    count=100  # Случайно перезаписали глобальную переменную!
}

increment
echo $count  # 1
other_function
echo $count  # 100 - неожиданный результат!
```

Это может привести к трудноотлавливаемым ошибкам, особенно в больших скриптах.

## Ключевое слово local

Команда `local` создаёт переменную, видимую только внутри функции:

```bash
my_function() {
    local my_var="локальное значение"
    echo "Внутри функции: $my_var"
}

my_var="глобальное значение"
my_function
echo "После функции: $my_var"  # глобальное значение
```

### Синтаксис local

```bash
# Объявление с инициализацией
local name="John"
local count=0

# Объявление без инициализации
local result

# Множественное объявление
local a=1 b=2 c=3

# Объявление с результатом команды
local current_date=$(date +%Y-%m-%d)
local files=$(ls -1)
```

## Область видимости

### Локальные переменные видны во вложенных вызовах

```bash
outer() {
    local message="Привет"

    inner() {
        echo "inner видит: $message"  # Работает!
    }

    inner
}

outer  # inner видит: Привет
```

### Переопределение в вложенной функции

```bash
level1() {
    local var="level1"
    echo "level1: $var"
    level2
    echo "level1 после level2: $var"  # Всё ещё level1
}

level2() {
    local var="level2"
    echo "level2: $var"
}

level1
```

Вывод:
```
level1: level1
level2: level2
level1 после level2: level1
```

## Передача переменных по ссылке

В bash 4.3+ можно использовать nameref для передачи по ссылке:

```bash
# Использование declare -n (nameref)
swap() {
    declare -n ref1=$1
    declare -n ref2=$2

    local temp=$ref1
    ref1=$ref2
    ref2=$temp
}

a=10
b=20
echo "До: a=$a, b=$b"  # До: a=10, b=20
swap a b
echo "После: a=$a, b=$b"  # После: a=20, b=10
```

### Возврат значения через nameref

```bash
get_max() {
    declare -n result=$1
    local a=$2
    local b=$3

    if [[ $a -gt $b ]]; then
        result=$a
    else
        result=$b
    fi
}

max_value=0
get_max max_value 15 23
echo "Максимум: $max_value"  # Максимум: 23
```

## Использование local с declare

`declare` внутри функции автоматически создаёт локальную переменную:

```bash
my_function() {
    declare my_var="локальная"
    declare -i my_int=42
    declare -a my_array=(1 2 3)

    echo $my_var
    echo $my_int
    echo ${my_array[@]}
}

my_function
echo $my_var  # Пусто - переменная была локальной
```

## Распространённые ошибки

### Ошибка 1: local в подстановке команды

```bash
# НЕПРАВИЛЬНО - код возврата теряется
my_function() {
    local result=$(some_command)  # Код возврата = 0 (от local)
    echo $?  # Всегда 0!
}

# ПРАВИЛЬНО - разделить объявление и присваивание
my_function() {
    local result
    result=$(some_command)
    echo $?  # Реальный код возврата some_command
}
```

### Ошибка 2: Использование local вне функции

```bash
# НЕПРАВИЛЬНО - local работает только в функции
local my_var="value"  # bash: local: can only be used in a function
```

### Ошибка 3: Затенение глобальной переменной

```bash
DEBUG=true

debug_print() {
    local DEBUG=false  # Случайно затенили глобальную
    # ...
}

if [[ $DEBUG == true ]]; then
    # Это работает, потому что вызывается вне функции
    echo "Debug mode"
fi
```

## Практические паттерны

### Паттерн: Сохранение и восстановление

```bash
with_changed_directory() {
    local original_dir=$(pwd)
    local target_dir=$1
    shift

    cd "$target_dir" || return 1

    # Выполняем команду
    "$@"
    local exit_code=$?

    # Возвращаемся в исходную директорию
    cd "$original_dir"

    return $exit_code
}

with_changed_directory /tmp ls -la
```

### Паттерн: Инкапсуляция состояния

```bash
# Счётчик с локальным состоянием через замыкание
make_counter() {
    local name=$1
    local count=0

    # Создаём функции, которые работают с локальным состоянием
    eval "
        ${name}_increment() { ((count++)); }
        ${name}_get() { echo \$count; }
        ${name}_reset() { count=0; }
    "
}

make_counter hits
hits_increment
hits_increment
hits_increment
echo "Счётчик: $(hits_get)"  # Счётчик: 3
```

### Паттерн: Безопасные временные файлы

```bash
process_with_temp() {
    local temp_file
    temp_file=$(mktemp)

    # Гарантированная очистка
    trap "rm -f '$temp_file'" RETURN

    # Работа с временным файлом
    echo "data" > "$temp_file"
    cat "$temp_file"

    # temp_file удалится автоматически при выходе из функции
}
```

## Best Practices

### 1. Всегда используйте local для переменных функции

```bash
# Хорошо
process_file() {
    local file=$1
    local content
    local line_count

    content=$(cat "$file")
    line_count=$(wc -l < "$file")
    # ...
}

# Плохо
process_file() {
    file=$1
    content=$(cat "$file")
    line_count=$(wc -l < "$file")
    # Все переменные глобальны и могут конфликтовать
}
```

### 2. Объявляйте переменные в начале функции

```bash
my_function() {
    # Объявления в начале
    local input=$1
    local output_file=$2
    local temp_dir="/tmp"
    local result
    local status

    # Логика функции
    result=$(process "$input")
    status=$?

    # ...
}
```

### 3. Разделяйте объявление и присваивание для критичных команд

```bash
fetch_data() {
    local url=$1
    local response
    local http_code

    # Разделяем, чтобы проверить код возврата
    response=$(curl -s "$url")
    http_code=$?

    if [[ $http_code -ne 0 ]]; then
        echo "Ошибка загрузки" >&2
        return 1
    fi

    echo "$response"
}
```

### 4. Используйте readonly для констант внутри функций

```bash
calculate_tax() {
    local amount=$1
    local -r TAX_RATE=0.20  # readonly локальная константа

    local tax
    tax=$(echo "$amount * $TAX_RATE" | bc)

    echo "$tax"
}
```

---

[prev: 01-functions](./01-functions.md) | [next: 03-script-structure](./03-script-structure.md)
