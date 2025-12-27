# Структура скрипта

## Проектирование сверху вниз (Top-Down Design)

**Top-Down Design** - это методология разработки, при которой сначала определяется общая структура программы, а затем детализируются отдельные компоненты.

### Принцип работы

1. Начните с главной функции, описывающей общий процесс
2. Разбейте на подзадачи (функции-заглушки)
3. Постепенно реализуйте каждую подзадачу
4. Повторяйте до достижения нужной детализации

### Пример: скрипт резервного копирования

**Шаг 1: Общая структура**

```bash
#!/bin/bash

main() {
    parse_arguments "$@"
    validate_configuration
    create_backup
    cleanup_old_backups
    send_notification
}

main "$@"
```

**Шаг 2: Функции-заглушки**

```bash
parse_arguments() {
    echo "TODO: Разбор аргументов"
}

validate_configuration() {
    echo "TODO: Проверка конфигурации"
}

create_backup() {
    echo "TODO: Создание резервной копии"
}

cleanup_old_backups() {
    echo "TODO: Удаление старых копий"
}

send_notification() {
    echo "TODO: Отправка уведомления"
}
```

**Шаг 3: Реализация функций**

```bash
parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -s|--source)
                SOURCE_DIR="$2"
                shift 2
                ;;
            -d|--dest)
                BACKUP_DIR="$2"
                shift 2
                ;;
            -h|--help)
                show_help
                exit 0
                ;;
            *)
                echo "Неизвестный аргумент: $1" >&2
                exit 1
                ;;
        esac
    done
}
```

## Рекомендуемая структура скрипта

```bash
#!/bin/bash
#
# Название: backup.sh
# Описание: Скрипт резервного копирования
# Автор: Имя Автора
# Дата: 2024-01-15
# Версия: 1.0.0
#
# Использование: backup.sh [опции] <источник>
#
# Опции:
#   -d, --dest DIR    Директория для бэкапов
#   -k, --keep N      Хранить N последних копий
#   -v, --verbose     Подробный вывод
#   -h, --help        Показать справку
#

# ============================================
# Настройки Shell
# ============================================
set -euo pipefail

# ============================================
# Константы
# ============================================
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
readonly SCRIPT_VERSION="1.0.0"

readonly DEFAULT_BACKUP_DIR="/var/backups"
readonly DEFAULT_KEEP_COUNT=7

# Цвета для вывода
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[0;33m'
readonly NC='\033[0m'

# ============================================
# Глобальные переменные
# ============================================
source_dir=""
backup_dir="$DEFAULT_BACKUP_DIR"
keep_count=$DEFAULT_KEEP_COUNT
verbose=false

# ============================================
# Функции: Утилиты
# ============================================

log_info() {
    echo -e "${GREEN}[INFO]${NC} $*"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $*" >&2
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $*" >&2
}

log_debug() {
    if [[ "$verbose" == true ]]; then
        echo -e "[DEBUG] $*"
    fi
}

die() {
    log_error "$@"
    exit 1
}

# ============================================
# Функции: Справка и версия
# ============================================

show_help() {
    cat << EOF
$SCRIPT_NAME v$SCRIPT_VERSION - Скрипт резервного копирования

Использование: $SCRIPT_NAME [опции] <источник>

Опции:
    -d, --dest DIR    Директория для бэкапов (по умолчанию: $DEFAULT_BACKUP_DIR)
    -k, --keep N      Хранить N последних копий (по умолчанию: $DEFAULT_KEEP_COUNT)
    -v, --verbose     Подробный вывод
    -h, --help        Показать эту справку
    --version         Показать версию

Примеры:
    $SCRIPT_NAME /var/www
    $SCRIPT_NAME -d /backup -k 14 /home/user
    $SCRIPT_NAME --verbose --dest /backup /etc
EOF
}

show_version() {
    echo "$SCRIPT_NAME версия $SCRIPT_VERSION"
}

# ============================================
# Функции: Разбор аргументов
# ============================================

parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -d|--dest)
                backup_dir="$2"
                shift 2
                ;;
            -k|--keep)
                keep_count="$2"
                shift 2
                ;;
            -v|--verbose)
                verbose=true
                shift
                ;;
            -h|--help)
                show_help
                exit 0
                ;;
            --version)
                show_version
                exit 0
                ;;
            --)
                shift
                break
                ;;
            -*)
                die "Неизвестная опция: $1"
                ;;
            *)
                source_dir="$1"
                shift
                ;;
        esac
    done

    # Оставшиеся аргументы
    if [[ -z "$source_dir" && $# -gt 0 ]]; then
        source_dir="$1"
    fi
}

# ============================================
# Функции: Валидация
# ============================================

validate_arguments() {
    if [[ -z "$source_dir" ]]; then
        die "Не указана директория для резервного копирования"
    fi

    if [[ ! -d "$source_dir" ]]; then
        die "Директория не существует: $source_dir"
    fi

    if [[ ! -r "$source_dir" ]]; then
        die "Нет прав на чтение: $source_dir"
    fi

    if ! [[ "$keep_count" =~ ^[0-9]+$ ]]; then
        die "Параметр --keep должен быть числом"
    fi
}

ensure_backup_dir() {
    if [[ ! -d "$backup_dir" ]]; then
        log_info "Создание директории: $backup_dir"
        mkdir -p "$backup_dir" || die "Не удалось создать: $backup_dir"
    fi
}

# ============================================
# Функции: Основная логика
# ============================================

create_backup() {
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local source_name=$(basename "$source_dir")
    local backup_file="${backup_dir}/${source_name}_${timestamp}.tar.gz"

    log_info "Создание резервной копии: $source_dir"
    log_debug "Файл: $backup_file"

    if tar -czf "$backup_file" -C "$(dirname "$source_dir")" "$source_name"; then
        log_info "Резервная копия создана: $backup_file"
        echo "$backup_file"
    else
        die "Ошибка при создании резервной копии"
    fi
}

cleanup_old_backups() {
    local source_name=$(basename "$source_dir")
    local pattern="${source_name}_*.tar.gz"

    log_debug "Поиск старых копий: $pattern"

    local backup_count
    backup_count=$(find "$backup_dir" -name "$pattern" -type f | wc -l)

    if [[ $backup_count -gt $keep_count ]]; then
        local to_delete=$((backup_count - keep_count))
        log_info "Удаление $to_delete старых копий"

        find "$backup_dir" -name "$pattern" -type f -printf '%T@ %p\n' | \
            sort -n | \
            head -n "$to_delete" | \
            cut -d' ' -f2- | \
            xargs rm -f

        log_debug "Старые копии удалены"
    else
        log_debug "Очистка не требуется (копий: $backup_count, лимит: $keep_count)"
    fi
}

# ============================================
# Функция main
# ============================================

main() {
    parse_arguments "$@"
    validate_arguments
    ensure_backup_dir
    create_backup
    cleanup_old_backups

    log_info "Готово!"
}

# ============================================
# Точка входа
# ============================================

# Запускаем main только если скрипт выполняется напрямую
# (не через source)
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

## Секции скрипта

### 1. Заголовок (Header)

```bash
#!/bin/bash
#
# Название: имя_скрипта.sh
# Описание: Краткое описание назначения
# Автор: Имя Автора
# Дата: YYYY-MM-DD
# Версия: X.Y.Z
#
```

### 2. Настройки Shell

```bash
set -euo pipefail
# -e: выход при ошибке
# -u: ошибка при использовании неопределённой переменной
# -o pipefail: ошибка в pipe приводит к ошибке всей команды
```

### 3. Константы

```bash
readonly SCRIPT_NAME=$(basename "$0")
readonly CONFIG_FILE="/etc/app.conf"
readonly MAX_RETRIES=3
```

### 4. Глобальные переменные

```bash
verbose=false
debug=false
output_file=""
```

### 5. Функции-утилиты

Общие функции: логирование, проверки, вывод ошибок.

### 6. Бизнес-логика

Функции, реализующие основную логику скрипта.

### 7. Функция main

Точка входа, объединяющая все части.

### 8. Запуск

```bash
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

Эта проверка позволяет использовать скрипт как библиотеку (через `source`).

## Принципы хорошей структуры

### Единственная ответственность

Каждая функция должна делать одну вещь:

```bash
# Плохо - функция делает слишком много
process_file() {
    validate_file "$1"
    parse_content "$1"
    transform_data
    save_output
    send_notification
}

# Хорошо - разделение на отдельные функции
main() {
    local file=$1
    validate_file "$file"
    local data=$(parse_content "$file")
    local result=$(transform_data "$data")
    save_output "$result"
    send_notification
}
```

### Минимизация глобальных переменных

```bash
# Плохо - много глобальных переменных
input_file=""
output_file=""
temp_dir=""
format=""
verbose=""

# Лучше - передавать как аргументы
process() {
    local input_file=$1
    local output_file=$2
    local format=$3
    # ...
}
```

### Ранний выход при ошибках

```bash
# Плохо - глубокая вложенность
process_file() {
    if [[ -f "$1" ]]; then
        if [[ -r "$1" ]]; then
            if [[ -s "$1" ]]; then
                # обработка
            fi
        fi
    fi
}

# Хорошо - ранний выход
process_file() {
    local file=$1

    [[ -f "$file" ]] || { echo "Не файл: $file" >&2; return 1; }
    [[ -r "$file" ]] || { echo "Нет доступа: $file" >&2; return 1; }
    [[ -s "$file" ]] || { echo "Файл пуст: $file" >&2; return 1; }

    # обработка
}
```
