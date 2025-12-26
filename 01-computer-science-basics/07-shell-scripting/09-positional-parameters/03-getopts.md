# Команда getopts

## Что такое getopts

`getopts` - встроенная команда bash для парсинга опций командной строки в стиле UNIX. Поддерживает короткие опции (-a, -b) с возможными аргументами.

## Базовый синтаксис

```bash
getopts optstring variable
```

- `optstring` - строка с допустимыми опциями
- `variable` - переменная для текущей опции

## Простой пример

```bash
#!/bin/bash

while getopts "abc" opt; do
    case $opt in
        a)
            echo "Опция -a"
            ;;
        b)
            echo "Опция -b"
            ;;
        c)
            echo "Опция -c"
            ;;
        ?)
            echo "Неизвестная опция"
            exit 1
            ;;
    esac
done
```

Запуск:
```bash
./script.sh -a -b -c    # Три отдельные опции
./script.sh -abc        # То же самое, объединённо
```

## Опции с аргументами

Двоеточие после буквы означает, что опция требует аргумент:

```bash
#!/bin/bash

while getopts "f:o:v" opt; do
    case $opt in
        f)
            input_file=$OPTARG
            echo "Входной файл: $input_file"
            ;;
        o)
            output_file=$OPTARG
            echo "Выходной файл: $output_file"
            ;;
        v)
            verbose=true
            echo "Включён подробный режим"
            ;;
        ?)
            echo "Использование: $0 [-v] -f input -o output"
            exit 1
            ;;
    esac
done
```

Запуск:
```bash
./script.sh -f input.txt -o output.txt -v
./script.sh -vf input.txt -o output.txt  # -v и -f вместе
```

## Специальные переменные

| Переменная | Описание |
|------------|----------|
| `$OPTARG` | Аргумент текущей опции |
| `$OPTIND` | Индекс следующего аргумента для обработки |
| `$opt` | Текущая опция (в переменной, указанной в getopts) |

## Тихий режим ошибок

Двоеточие в начале optstring включает тихий режим:

```bash
#!/bin/bash

# Без двоеточия в начале - getopts выводит ошибки сам
while getopts "f:o:" opt; do
    # ...
done

# С двоеточием - мы обрабатываем ошибки сами
while getopts ":f:o:" opt; do
    case $opt in
        f)
            file=$OPTARG
            ;;
        :)
            echo "Опция -$OPTARG требует аргумент"
            exit 1
            ;;
        ?)
            echo "Неизвестная опция: -$OPTARG"
            exit 1
            ;;
    esac
done
```

## Обработка оставшихся аргументов

```bash
#!/bin/bash

while getopts "f:v" opt; do
    case $opt in
        f) file=$OPTARG ;;
        v) verbose=true ;;
    esac
done

# Сдвигаем обработанные опции
shift $((OPTIND - 1))

# Теперь $@ содержит оставшиеся аргументы (не опции)
echo "Файлы для обработки: $@"

for file in "$@"; do
    echo "Обработка: $file"
done
```

Запуск:
```bash
./script.sh -v -f config.txt file1.txt file2.txt
# Опции: -v, -f config.txt
# Оставшиеся: file1.txt file2.txt
```

## Полный пример скрипта

```bash
#!/bin/bash

# Справка
usage() {
    cat << EOF
Использование: $(basename "$0") [опции] файл...

Опции:
    -h          Показать эту справку
    -v          Подробный вывод
    -q          Тихий режим
    -o FILE     Выходной файл
    -n NUM      Количество строк (по умолчанию: 10)
    -d CHAR     Разделитель (по умолчанию: ,)

Примеры:
    $(basename "$0") -v -n 20 input.txt
    $(basename "$0") -o output.csv -d ";" input.txt
EOF
    exit "${1:-0}"
}

# Значения по умолчанию
verbose=false
quiet=false
output=""
num_lines=10
delimiter=","

# Парсинг опций
while getopts ":hqvo:n:d:" opt; do
    case $opt in
        h)
            usage 0
            ;;
        v)
            verbose=true
            ;;
        q)
            quiet=true
            ;;
        o)
            output=$OPTARG
            ;;
        n)
            if [[ "$OPTARG" =~ ^[0-9]+$ ]]; then
                num_lines=$OPTARG
            else
                echo "Ошибка: -n требует числовой аргумент" >&2
                exit 1
            fi
            ;;
        d)
            delimiter=$OPTARG
            ;;
        :)
            echo "Ошибка: опция -$OPTARG требует аргумент" >&2
            usage 1
            ;;
        ?)
            echo "Ошибка: неизвестная опция -$OPTARG" >&2
            usage 1
            ;;
    esac
done

# Сдвигаем обработанные опции
shift $((OPTIND - 1))

# Проверяем конфликты
if $verbose && $quiet; then
    echo "Ошибка: -v и -q несовместимы" >&2
    exit 1
fi

# Проверяем наличие файлов
if [[ $# -eq 0 ]]; then
    echo "Ошибка: не указаны файлы" >&2
    usage 1
fi

# Основная логика
$verbose && echo "Настройки: verbose=$verbose, output=$output, lines=$num_lines"

for file in "$@"; do
    $verbose && echo "Обработка: $file"

    if [[ ! -f "$file" ]]; then
        $quiet || echo "Предупреждение: файл не найден: $file" >&2
        continue
    fi

    # ... обработка файла ...
done

$verbose && echo "Готово!"
```

## Ограничения getopts

1. **Только короткие опции** - не поддерживает --long-option
2. **Порядок важен** - опции должны идти перед позиционными аргументами
3. **Нет автогенерации справки** - нужно писать вручную

## Альтернатива: getopt (внешняя команда)

Для длинных опций используйте `getopt` (не путать с `getopts`):

```bash
#!/bin/bash

# GNU getopt с длинными опциями
OPTS=$(getopt -o vhf:o: --long verbose,help,file:,output: -n "$0" -- "$@")
if [[ $? -ne 0 ]]; then
    exit 1
fi

eval set -- "$OPTS"

while true; do
    case $1 in
        -v|--verbose)
            verbose=true
            shift
            ;;
        -h|--help)
            echo "Справка"
            exit 0
            ;;
        -f|--file)
            file=$2
            shift 2
            ;;
        -o|--output)
            output=$2
            shift 2
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Ошибка парсинга"
            exit 1
            ;;
    esac
done

echo "Файлы: $@"
```

## Сброс OPTIND

При повторном использовании getopts (например, в функции) нужно сбросить OPTIND:

```bash
parse_args() {
    local OPTIND  # Локальный OPTIND
    while getopts "ab" opt; do
        case $opt in
            a) echo "a" ;;
            b) echo "b" ;;
        esac
    done
}

# Или
OPTIND=1  # Сброс перед повторным использованием
```
