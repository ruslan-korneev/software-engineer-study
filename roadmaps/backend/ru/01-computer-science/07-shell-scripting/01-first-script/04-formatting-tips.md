# Советы по форматированию

## Читаемость кода

Shell-скрипты должны быть читаемыми не только для компьютера, но и для людей (включая вас через полгода).

### Отступы

Используйте отступы для обозначения вложенности:

```bash
# Хорошо - с отступами
if [[ -f "$file" ]]; then
    if [[ -r "$file" ]]; then
        echo "Файл существует и доступен для чтения"
        cat "$file"
    fi
fi

# Плохо - без отступов
if [[ -f "$file" ]]; then
if [[ -r "$file" ]]; then
echo "Файл существует и доступен для чтения"
cat "$file"
fi
fi
```

Рекомендации:
- Используйте 4 пробела или 1 табуляцию
- Будьте последовательны во всем скрипте
- Многие предпочитают пробелы (они одинаково отображаются везде)

### Пустые строки

Разделяйте логические блоки пустыми строками:

```bash
#!/bin/bash

# Переменные
SOURCE_DIR="/var/log"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d)

# Функции
create_backup() {
    tar -czf "$BACKUP_DIR/logs_$DATE.tar.gz" "$SOURCE_DIR"
}

cleanup_old() {
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete
}

# Основной код
create_backup
cleanup_old
echo "Готово!"
```

## Именование

### Переменные

```bash
# Хорошо - описательные имена
backup_directory="/backup"
max_retry_count=3
is_verbose=true

# Плохо - непонятные сокращения
bd="/backup"
mrc=3
v=true
```

Соглашения:
- `snake_case` для локальных переменных: `my_variable`
- `UPPER_CASE` для констант и переменных окружения: `MAX_RETRIES`
- Избегайте однобуквенных имен (кроме счетчиков циклов `i`, `j`)

### Функции

```bash
# Хорошо - глаголы, описывающие действие
create_backup()
send_notification()
validate_input()
get_user_name()

# Плохо - непонятные имена
backup()
proc()
do_it()
```

## Кавычки

### Всегда используйте кавычки для переменных

```bash
# Хорошо
echo "$variable"
cp "$source" "$destination"

# Плохо - проблемы с пробелами и спецсимволами
echo $variable
cp $source $destination
```

### Исключения

Кавычки можно опустить:
- Внутри `[[ ]]` (но лучше все равно использовать)
- В присваивании: `var=$other_var`
- При использовании `@` для массивов внутри `"${array[@]}"`

### Одинарные vs двойные кавычки

```bash
name="World"

# Двойные - переменные раскрываются
echo "Hello, $name"   # Hello, World

# Одинарные - всё буквально
echo 'Hello, $name'   # Hello, $name

# Обратные - выполнение команды (устаревший синтаксис)
echo "Today is `date`"

# Современный синтаксис - $()
echo "Today is $(date)"
```

## Длинные строки

### Перенос с backslash

```bash
# Длинная команда
rsync -avz \
    --progress \
    --exclude='*.tmp' \
    --exclude='*.log' \
    /source/ \
    /destination/
```

### Массивы для сложных команд

```bash
# Вместо очень длинной строки
curl_opts=(
    --silent
    --show-error
    --location
    --header "Content-Type: application/json"
    --header "Authorization: Bearer $TOKEN"
)

curl "${curl_opts[@]}" "$url"
```

## Комментарии

### Когда комментировать

```bash
# Хорошо - объясняет ПОЧЕМУ
# Используем sleep, чтобы дать сервису время на инициализацию
sleep 5

# Плохо - объясняет ЧТО (и так видно из кода)
# Ждём 5 секунд
sleep 5
```

### Заголовки разделов

```bash
#!/bin/bash

#######################################
# Константы
#######################################
readonly CONFIG_FILE="/etc/app/config"
readonly LOG_FILE="/var/log/app.log"

#######################################
# Функции
#######################################

# Выводит сообщение об ошибке в stderr
# Arguments:
#   $1 - текст сообщения
# Returns:
#   None
log_error() {
    echo "[ERROR] $1" >&2
}
```

## Структура и организация

### Порядок элементов

Рекомендуемый порядок в скрипте:

1. Shebang
2. Описание скрипта (в комментариях)
3. Настройки shell (`set` команды)
4. Константы
5. Глобальные переменные
6. Функции
7. Основной код

```bash
#!/bin/bash
#
# Скрипт для развертывания приложения
# Использование: deploy.sh [environment]
#

set -euo pipefail

# Константы
readonly DEPLOY_DIR="/opt/app"
readonly DEFAULT_ENV="production"

# Переменные
environment="${1:-$DEFAULT_ENV}"

# Функции
deploy() {
    echo "Deploying to $1..."
}

# Основной код
deploy "$environment"
```

### Функция main

Выносите основной код в функцию `main`:

```bash
#!/bin/bash

setup() {
    # подготовка
    :
}

process() {
    # обработка
    :
}

cleanup() {
    # очистка
    :
}

main() {
    setup
    process
    cleanup
}

# Вызов main с передачей всех аргументов
main "$@"
```

## Рекомендуемые настройки

### set команды для надежности

```bash
#!/bin/bash
set -euo pipefail

# -e : выход при ошибке любой команды
# -u : ошибка при использовании неопределённой переменной
# -o pipefail : ошибка, если любая команда в pipe завершилась с ошибкой
```

Подробнее об этих настройках - в разделе "Debugging".

## Инструменты для проверки стиля

### ShellCheck

[ShellCheck](https://www.shellcheck.net/) - статический анализатор shell-скриптов:

```bash
# Установка
sudo apt install shellcheck  # Debian/Ubuntu
brew install shellcheck      # macOS

# Использование
shellcheck script.sh
```

ShellCheck находит:
- Синтаксические ошибки
- Типичные ошибки (пропущенные кавычки и т.д.)
- Проблемы совместимости
- Стилистические проблемы

### shfmt

Форматтер для shell-скриптов:

```bash
# Установка
brew install shfmt  # macOS

# Использование
shfmt -w script.sh  # Форматировать на месте
shfmt -d script.sh  # Показать diff
```
