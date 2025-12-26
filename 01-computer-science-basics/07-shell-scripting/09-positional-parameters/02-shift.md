# Команда shift

## Что такое shift

Команда `shift` сдвигает позиционные параметры влево:
- `$2` становится `$1`
- `$3` становится `$2`
- И так далее
- `$1` удаляется
- `$#` уменьшается на 1

## Базовое использование

```bash
#!/bin/bash

echo "До shift:"
echo "\$1 = $1"
echo "\$2 = $2"
echo "\$3 = $3"
echo "\$# = $#"

shift

echo ""
echo "После shift:"
echo "\$1 = $1"
echo "\$2 = $2"
echo "\$3 = $3"
echo "\$# = $#"
```

Запуск: `./script.sh a b c d`

Вывод:
```
До shift:
$1 = a
$2 = b
$3 = c
$# = 4

После shift:
$1 = b
$2 = c
$3 = d
$# = 3
```

## shift N

Можно сдвинуть на несколько позиций:

```bash
#!/bin/bash

echo "Аргументы: $@"

shift 2  # Сдвинуть на 2 позиции

echo "После shift 2: $@"
```

Запуск: `./script.sh a b c d e`

```
Аргументы: a b c d e
После shift 2: c d e
```

## Обработка аргументов в цикле

### Простой цикл

```bash
#!/bin/bash

while [[ $# -gt 0 ]]; do
    echo "Обработка: $1"
    shift
done
```

### Обработка опций

```bash
#!/bin/bash

while [[ $# -gt 0 ]]; do
    case $1 in
        -v|--verbose)
            verbose=true
            shift
            ;;
        -o|--output)
            output_file=$2
            shift 2
            ;;
        -h|--help)
            echo "Справка..."
            exit 0
            ;;
        --)
            shift
            break  # Всё после -- считается аргументами
            ;;
        -*)
            echo "Неизвестная опция: $1"
            exit 1
            ;;
        *)
            break  # Первый не-опциональный аргумент
            ;;
    esac
done

# Оставшиеся аргументы
echo "Оставшиеся аргументы: $@"
```

## Практические примеры

### Скрипт с опциями

```bash
#!/bin/bash

# Значения по умолчанию
verbose=false
dry_run=false
output=""
files=()

# Парсинг опций
while [[ $# -gt 0 ]]; do
    case $1 in
        -v|--verbose)
            verbose=true
            shift
            ;;
        -n|--dry-run)
            dry_run=true
            shift
            ;;
        -o|--output)
            if [[ -n "$2" && ! "$2" =~ ^- ]]; then
                output=$2
                shift 2
            else
                echo "Ошибка: -o требует аргумент"
                exit 1
            fi
            ;;
        -h|--help)
            echo "Использование: $0 [опции] файлы..."
            echo "  -v, --verbose   Подробный вывод"
            echo "  -n, --dry-run   Тестовый запуск"
            echo "  -o, --output    Выходной файл"
            exit 0
            ;;
        --)
            shift
            break
            ;;
        -*)
            echo "Неизвестная опция: $1"
            exit 1
            ;;
        *)
            files+=("$1")
            shift
            ;;
    esac
done

# Добавляем оставшиеся аргументы в файлы
files+=("$@")

# Вывод результата парсинга
echo "verbose=$verbose"
echo "dry_run=$dry_run"
echo "output=$output"
echo "files=${files[*]}"
```

### Обработка переменного числа аргументов

```bash
#!/bin/bash

# Первый аргумент - операция, остальные - данные
operation=$1
shift

case $operation in
    sum)
        total=0
        while [[ $# -gt 0 ]]; do
            ((total += $1))
            shift
        done
        echo "Сумма: $total"
        ;;
    max)
        max=$1
        shift
        while [[ $# -gt 0 ]]; do
            [[ $1 -gt $max ]] && max=$1
            shift
        done
        echo "Максимум: $max"
        ;;
    *)
        echo "Неизвестная операция: $operation"
        ;;
esac
```

### Разделение опций и файлов

```bash
#!/bin/bash

# Опции в начале, потом файлы
options=()
files=()

while [[ $# -gt 0 ]]; do
    case $1 in
        -*)
            options+=("$1")
            shift
            ;;
        *)
            # Как только встретили не-опцию, всё остальное - файлы
            files=("$@")
            break
            ;;
    esac
done

echo "Опции: ${options[*]}"
echo "Файлы: ${files[*]}"
```

### Обработка подкоманд

```bash
#!/bin/bash
# git-подобный интерфейс: script.sh command [options] [args]

command=$1
shift

case $command in
    add)
        # Обработка опций для add
        force=false
        while [[ $# -gt 0 ]]; do
            case $1 in
                -f|--force) force=true; shift ;;
                *) break ;;
            esac
        done
        echo "ADD: force=$force, files=$*"
        ;;

    remove)
        recursive=false
        while [[ $# -gt 0 ]]; do
            case $1 in
                -r|--recursive) recursive=true; shift ;;
                *) break ;;
            esac
        done
        echo "REMOVE: recursive=$recursive, files=$*"
        ;;

    *)
        echo "Usage: $0 {add|remove} [options] [files]"
        ;;
esac
```

## Сохранение аргументов перед shift

```bash
#!/bin/bash

# Сохраняем все оригинальные аргументы
original_args=("$@")

# Первый аргумент - команда
command=$1
shift

# Обработка...
echo "Команда: $command"
echo "Оставшиеся: $@"

# Можно восстановить
set -- "${original_args[@]}"
echo "Восстановлено: $@"
```

## Ошибки при shift

```bash
#!/bin/bash

# shift больше, чем есть аргументов
echo "Аргументов: $#"
shift 10  # Если аргументов < 10, ничего не произойдёт в bash
# В POSIX shell это ошибка

# Безопасный shift
safe_shift() {
    local n=${1:-1}
    if [[ $# -ge $n ]]; then
        shift $n
    fi
}
```

## shift в функциях

```bash
#!/bin/bash

process_args() {
    local prefix=$1
    shift  # Влияет только на аргументы функции

    for arg in "$@"; do
        echo "$prefix: $arg"
    done
}

# Аргументы скрипта остаются нетронутыми
process_args "Item" "a" "b" "c"
echo "Аргументы скрипта: $@"  # Не изменились
```
