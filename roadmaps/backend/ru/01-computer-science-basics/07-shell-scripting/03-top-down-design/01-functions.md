# Функции в Bash

## Что такое функции

Функции в bash - это именованные блоки кода, которые можно вызывать многократно. Они позволяют:
- Избежать дублирования кода
- Разбить скрипт на логические части
- Улучшить читаемость и поддерживаемость
- Реализовать абстракции

## Синтаксис объявления

### Два способа объявления

```bash
# Способ 1: с ключевым словом function
function greet {
    echo "Привет!"
}

# Способ 2: без ключевого слова (POSIX-совместимый)
greet() {
    echo "Привет!"
}

# Вызов функции
greet
```

**Рекомендуется** использовать второй способ (без `function`) для лучшей совместимости.

### Однострочные функции

```bash
# Точка с запятой перед закрывающей скобкой обязательна
say_hello() { echo "Hello, $1!"; }

# Вызов
say_hello "World"  # Hello, World!
```

## Аргументы функций

Функции получают аргументы так же, как скрипты - через позиционные параметры:

```bash
greet() {
    local name=$1     # Первый аргумент
    local time=$2     # Второй аргумент

    echo "Доброе $time, $name!"
}

greet "Иван" "утро"   # Доброе утро, Иван!
greet "Мария" "день"  # Доброе день, Мария!
```

### Все позиционные параметры в функции

```bash
show_args() {
    echo "Имя функции: $FUNCNAME"
    echo "Количество аргументов: $#"
    echo "Первый аргумент: $1"
    echo "Все аргументы: $@"

    # Перебор всех аргументов
    for arg in "$@"; do
        echo "- $arg"
    done
}

show_args "один" "два" "три"
```

### Значения по умолчанию

```bash
greet() {
    local name=${1:-"Гость"}
    local greeting=${2:-"Привет"}

    echo "$greeting, $name!"
}

greet              # Привет, Гость!
greet "Иван"       # Привет, Иван!
greet "Иван" "Здравствуй"  # Здравствуй, Иван!
```

## Возврат значений

### Код возврата (return)

Функции могут возвращать числовой код (0-255):

```bash
is_file_exists() {
    local file=$1

    if [[ -f "$file" ]]; then
        return 0  # Успех (файл существует)
    else
        return 1  # Ошибка (файл не существует)
    fi
}

# Использование
if is_file_exists "/etc/passwd"; then
    echo "Файл существует"
else
    echo "Файл не найден"
fi

# Или проверка $?
is_file_exists "/etc/passwd"
echo "Код возврата: $?"
```

### Вывод результата (echo)

Для возврата данных (строк, чисел) используйте вывод:

```bash
get_username() {
    whoami
}

add_numbers() {
    local a=$1
    local b=$2
    echo $((a + b))
}

# Захват результата
current_user=$(get_username)
sum=$(add_numbers 5 3)

echo "Пользователь: $current_user"
echo "Сумма: $sum"
```

### Комбинация return и echo

```bash
divide() {
    local a=$1
    local b=$2

    if [[ $b -eq 0 ]]; then
        echo "Ошибка: деление на ноль" >&2
        return 1
    fi

    echo $((a / b))
    return 0
}

# Использование
result=$(divide 10 2)
if [[ $? -eq 0 ]]; then
    echo "Результат: $result"
else
    echo "Операция не удалась"
fi
```

## Область видимости

### Глобальные переменные

По умолчанию переменные в функции глобальны:

```bash
set_global() {
    my_var="Я глобальная"
}

my_var="Начальное значение"
set_global
echo "$my_var"  # Я глобальная
```

### Локальные переменные

Используйте `local` для создания локальных переменных:

```bash
set_local() {
    local my_var="Я локальная"
    echo "Внутри функции: $my_var"
}

my_var="Начальное значение"
set_local
echo "После функции: $my_var"  # Начальное значение
```

Подробнее о локальных переменных в следующем разделе.

## Вложенные функции

Функции можно определять внутри других функций:

```bash
outer() {
    inner() {
        echo "Я внутренняя функция"
    }

    echo "Я внешняя функция"
    inner
}

outer
inner  # Тоже работает! Функция определена после вызова outer
```

**Примечание:** Внутренняя функция становится доступной глобально после выполнения внешней.

## Рекурсия

Функции могут вызывать сами себя:

```bash
factorial() {
    local n=$1

    if [[ $n -le 1 ]]; then
        echo 1
    else
        local prev=$(factorial $((n - 1)))
        echo $((n * prev))
    fi
}

echo "5! = $(factorial 5)"  # 5! = 120
```

**Осторожно:** bash не оптимизирует хвостовую рекурсию, глубокая рекурсия может привести к переполнению стека.

## Практические примеры

### Логирование

```bash
log() {
    local level=$1
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    echo "[$timestamp] [$level] $message"
}

log_info() { log "INFO" "$@"; }
log_warn() { log "WARN" "$@"; }
log_error() { log "ERROR" "$@" >&2; }

log_info "Скрипт запущен"
log_warn "Диск заполнен на 80%"
log_error "Не удалось подключиться к БД"
```

### Проверка зависимостей

```bash
require_command() {
    local cmd=$1

    if ! command -v "$cmd" &> /dev/null; then
        echo "Ошибка: команда '$cmd' не найдена" >&2
        return 1
    fi
    return 0
}

check_dependencies() {
    local deps=("curl" "jq" "git")

    for dep in "${deps[@]}"; do
        if ! require_command "$dep"; then
            exit 1
        fi
    done

    echo "Все зависимости установлены"
}

check_dependencies
```

### Безопасное выполнение

```bash
run_safe() {
    local cmd="$*"

    echo "Выполняю: $cmd"

    if ! eval "$cmd"; then
        echo "Ошибка при выполнении: $cmd" >&2
        return 1
    fi

    return 0
}

run_safe ls -la /tmp
run_safe cat /nonexistent/file
```

### Повторные попытки

```bash
retry() {
    local max_attempts=${1:-3}
    local delay=${2:-5}
    shift 2
    local cmd="$*"
    local attempt=1

    while [[ $attempt -le $max_attempts ]]; do
        echo "Попытка $attempt из $max_attempts: $cmd"

        if eval "$cmd"; then
            return 0
        fi

        echo "Неудача. Ожидание $delay секунд..."
        sleep "$delay"
        ((attempt++))
    done

    echo "Все попытки исчерпаны" >&2
    return 1
}

retry 3 2 curl -s http://example.com
```

## Экспорт функций

Функции можно экспортировать для использования в подпроцессах:

```bash
my_function() {
    echo "Я экспортированная функция"
}

export -f my_function

# Теперь функция доступна в подпроцессах
bash -c 'my_function'
```

## Best Practices

1. **Используйте local** для переменных внутри функций
2. **Документируйте функции** комментариями
3. **Проверяйте аргументы** в начале функции
4. **Используйте говорящие имена** - глаголы для действий
5. **Следуйте принципу единственной ответственности**

```bash
# Пример хорошо оформленной функции
# Создаёт резервную копию файла
# Аргументы:
#   $1 - путь к исходному файлу
# Возвращает:
#   0 - успех
#   1 - файл не существует
#   2 - ошибка копирования
backup_file() {
    local source_file=$1
    local backup_dir="${BACKUP_DIR:-/var/backups}"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${backup_dir}/$(basename "$source_file").${timestamp}"

    # Проверка аргументов
    if [[ -z "$source_file" ]]; then
        echo "Ошибка: не указан файл" >&2
        return 1
    fi

    if [[ ! -f "$source_file" ]]; then
        echo "Ошибка: файл '$source_file' не существует" >&2
        return 1
    fi

    # Создание резервной копии
    if ! cp "$source_file" "$backup_file"; then
        echo "Ошибка при копировании" >&2
        return 2
    fi

    echo "Создана резервная копия: $backup_file"
    return 0
}
```
