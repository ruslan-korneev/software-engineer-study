# Трассировка выполнения

## Режим трассировки (set -x)

Режим трассировки выводит каждую команду перед выполнением:

```bash
#!/bin/bash
set -x  # Включить трассировку

name="World"
echo "Hello, $name"
```

Вывод:
```
+ name=World
+ echo 'Hello, World'
Hello, World
```

## Включение и отключение трассировки

### Глобально

```bash
#!/bin/bash
set -x  # Включить для всего скрипта
```

### Локально

```bash
#!/bin/bash

echo "Без трассировки"

set -x
echo "С трассировкой"
set +x

echo "Снова без трассировки"
```

### При запуске

```bash
bash -x script.sh
```

## Переменная PS4

`PS4` определяет префикс для строк трассировки:

```bash
#!/bin/bash

# По умолчанию PS4='+ '
# Можно добавить полезную информацию:

export PS4='+ ${BASH_SOURCE}:${LINENO}: ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x

my_function() {
    local x=10
    echo "x = $x"
}

my_function
```

Вывод:
```
+ script.sh:9: my_function(): local x=10
+ script.sh:10: my_function(): echo 'x = 10'
x = 10
```

### Полезные PS4 варианты

```bash
# С номером строки
export PS4='+(${LINENO}): '

# С именем скрипта и номером строки
export PS4='+${BASH_SOURCE}:${LINENO}: '

# С временной меткой
export PS4='+$(date +%T) '

# Полная информация
export PS4='+[${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]:-main}] '
```

## Режим verbose (set -v)

Выводит строки скрипта перед обработкой (до раскрытия переменных):

```bash
#!/bin/bash
set -v

name="World"
echo "Hello, $name"
```

Вывод:
```
name="World"
echo "Hello, $name"
Hello, World
```

## Сравнение -x и -v

```bash
#!/bin/bash

name="test"
echo "Value: $name"
```

С `set -v` (verbose):
```
name="test"
echo "Value: $name"
Value: test
```

С `set -x` (trace):
```
+ name=test
+ echo 'Value: test'
Value: test
```

С `set -xv` (оба):
```
name="test"
+ name=test
echo "Value: $name"
+ echo 'Value: test'
Value: test
```

## Перенаправление вывода трассировки

### В файл

```bash
#!/bin/bash
exec 2>/tmp/debug.log  # stderr в файл
set -x

echo "This goes to stdout"
# Трассировка идёт в /tmp/debug.log
```

### В отдельный файл (bash 4.1+)

```bash
#!/bin/bash
exec 3>/tmp/trace.log
BASH_XTRACEFD=3
set -x

echo "Normal output"
# Трассировка идёт в /tmp/trace.log
```

## Встроенные отладочные функции

### Функция debug

```bash
#!/bin/bash

DEBUG=${DEBUG:-false}

debug() {
    if $DEBUG; then
        echo "[DEBUG] $*" >&2
    fi
}

debug "Начало скрипта"
debug "Переменная x = $x"

# Запуск с отладкой:
# DEBUG=true ./script.sh
```

### Уровни логирования

```bash
#!/bin/bash

LOG_LEVEL=${LOG_LEVEL:-INFO}

log() {
    local level=$1
    shift
    local message="$*"

    # Порядок уровней: DEBUG < INFO < WARN < ERROR
    case $LOG_LEVEL in
        DEBUG)
            echo "[$(date +%H:%M:%S)] [$level] $message" >&2
            ;;
        INFO)
            [[ "$level" != "DEBUG" ]] && echo "[$(date +%H:%M:%S)] [$level] $message" >&2
            ;;
        WARN)
            [[ "$level" =~ ^(WARN|ERROR)$ ]] && echo "[$(date +%H:%M:%S)] [$level] $message" >&2
            ;;
        ERROR)
            [[ "$level" == "ERROR" ]] && echo "[$(date +%H:%M:%S)] [$level] $message" >&2
            ;;
    esac
}

log DEBUG "Отладочное сообщение"
log INFO "Информационное сообщение"
log WARN "Предупреждение"
log ERROR "Ошибка"
```

## trap для отладки

### DEBUG trap

```bash
#!/bin/bash

trap 'echo "Выполняется: $BASH_COMMAND"' DEBUG

x=1
y=2
z=$((x + y))
echo "z = $z"
```

Вывод:
```
Выполняется: x=1
Выполняется: y=2
Выполняется: z=$((x + y))
Выполняется: echo "z = $z"
z = 3
```

### Отслеживание переменных

```bash
#!/bin/bash

declare -t watched_var  # Установить trace атрибут

trap 'echo "watched_var изменена на: $watched_var"' DEBUG

watched_var=10
watched_var=20
watched_var=30
```

### ERR trap

```bash
#!/bin/bash
set -e

trap 'echo "Ошибка в строке $LINENO: $BASH_COMMAND"' ERR

echo "Начало"
false  # Эта команда вернёт ошибку
echo "Этого не будет"
```

## Пошаговое выполнение

### Интерактивная отладка с read

```bash
#!/bin/bash

STEP_DEBUG=${STEP_DEBUG:-false}

step() {
    if $STEP_DEBUG; then
        echo ">>> $*" >&2
        read -p "Press Enter to continue..." -r
    fi
}

step "Инициализация переменных"
x=10
y=20

step "Вычисление суммы"
sum=$((x + y))

step "Вывод результата"
echo "Сумма: $sum"
```

```bash
# Запуск в пошаговом режиме
STEP_DEBUG=true ./script.sh
```

## Профилирование

### Измерение времени выполнения

```bash
#!/bin/bash

TIMEFORMAT='Время: %R секунд'

time {
    # Код для измерения
    sleep 1
    for i in {1..1000}; do
        echo $i > /dev/null
    done
}
```

### Профилирование с PS4

```bash
#!/bin/bash

PS4='+ $(date +%s.%N) '
set -x

sleep 0.1
echo "Step 1"
sleep 0.2
echo "Step 2"
```

### Функция для профилирования

```bash
#!/bin/bash

profile_start() {
    _PROFILE_START=$(date +%s.%N)
}

profile_end() {
    local end=$(date +%s.%N)
    local duration=$(echo "$end - $_PROFILE_START" | bc)
    echo "Время выполнения: ${duration}s" >&2
}

profile_start
# Код для измерения
sleep 1
profile_end
```

## Практический пример отладки

```bash
#!/bin/bash

# Конфигурация отладки
DEBUG=${DEBUG:-false}
TRACE=${TRACE:-false}

# Настройка трассировки
if $TRACE; then
    export PS4='+[${BASH_SOURCE}:${LINENO}] '
    set -x
fi

# Функция отладки
debug() {
    $DEBUG && echo "[DEBUG] $*" >&2
}

# ERR trap для отлова ошибок
trap 'echo "ОШИБКА в строке $LINENO: $BASH_COMMAND" >&2' ERR

# Основной код
main() {
    debug "Вход в main()"

    local input=$1
    debug "input = '$input'"

    if [[ -z "$input" ]]; then
        echo "Ошибка: не указан ввод" >&2
        return 1
    fi

    process "$input"

    debug "Выход из main()"
}

process() {
    debug "Обработка: $1"
    echo "Результат: $1"
}

# Запуск
main "$@"
```

```bash
# Обычный запуск
./script.sh "test"

# С отладкой
DEBUG=true ./script.sh "test"

# С трассировкой
TRACE=true ./script.sh "test"

# С отладкой и трассировкой
DEBUG=true TRACE=true ./script.sh "test"
```
