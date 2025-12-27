# Константы в Bash

## Что такое константы

Константы - это переменные, значения которых не должны изменяться после инициализации. В bash нет настоящих констант как в компилируемых языках, но есть способы создать переменные "только для чтения".

## Команда readonly

### Создание констант

```bash
# Объявление константы
readonly PI=3.14159
readonly APP_NAME="MyApp"
readonly VERSION="1.0.0"

# Попытка изменить вызовет ошибку
PI=3.14  # bash: PI: readonly variable
```

### Readonly для существующей переменной

```bash
config_file="/etc/app.conf"
# ... какая-то логика инициализации ...
readonly config_file  # Теперь нельзя изменить

config_file="/other/path"  # Ошибка!
```

## Команда declare -r

Альтернативный способ создания констант:

```bash
declare -r DATABASE_HOST="localhost"
declare -r DATABASE_PORT=5432

# Комбинация с другими флагами
declare -ri MAX_CONNECTIONS=100  # readonly integer
declare -ra ALLOWED_HOSTS=("host1" "host2" "host3")  # readonly array
```

## Соглашения об именовании

По соглашению, константы пишутся в верхнем регистре:

```bash
# Константы - UPPER_CASE
readonly MAX_RETRIES=3
readonly DEFAULT_TIMEOUT=30
readonly CONFIG_DIR="/etc/myapp"

# Обычные переменные - lower_case или snake_case
current_retry=0
timeout=$DEFAULT_TIMEOUT
```

## Практическое применение

### Конфигурация скрипта

```bash
#!/bin/bash

# ==========================================
# Константы (не изменять после инициализации)
# ==========================================
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
readonly LOG_FILE="/var/log/${SCRIPT_NAME}.log"

# Коды выхода
readonly EXIT_SUCCESS=0
readonly EXIT_ERROR=1
readonly EXIT_INVALID_ARGS=2

# Настройки
readonly DEFAULT_CONFIG="${SCRIPT_DIR}/config.conf"
readonly BACKUP_DIR="/var/backups"
readonly MAX_BACKUP_AGE=30  # дней

# ==========================================
# Переменные (могут изменяться)
# ==========================================
config_file="${1:-$DEFAULT_CONFIG}"
verbose=false
```

### Цвета для вывода

```bash
#!/bin/bash

# Цвета (ANSI escape codes)
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[0;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m'  # No Color

# Использование
echo -e "${RED}Ошибка:${NC} Файл не найден"
echo -e "${GREEN}Успех:${NC} Операция завершена"
echo -e "${YELLOW}Предупреждение:${NC} Диск заполнен на 80%"
```

### Регулярные выражения

```bash
#!/bin/bash

# Паттерны валидации
readonly EMAIL_PATTERN='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
readonly IP_PATTERN='^([0-9]{1,3}\.){3}[0-9]{1,3}$'
readonly DATE_PATTERN='^[0-9]{4}-[0-9]{2}-[0-9]{2}$'

# Использование
validate_email() {
    if [[ $1 =~ $EMAIL_PATTERN ]]; then
        echo "Email валиден"
        return 0
    else
        echo "Email невалиден"
        return 1
    fi
}

validate_email "user@example.com"
```

## Группы констант

### Использование префиксов

```bash
#!/bin/bash

# Константы базы данных
readonly DB_HOST="localhost"
readonly DB_PORT=5432
readonly DB_NAME="myapp"
readonly DB_USER="appuser"

# Константы API
readonly API_BASE_URL="https://api.example.com"
readonly API_VERSION="v1"
readonly API_TIMEOUT=30

# Константы файловой системы
readonly FS_DATA_DIR="/var/data"
readonly FS_TEMP_DIR="/tmp/myapp"
readonly FS_LOG_DIR="/var/log/myapp"
```

### Ассоциативные массивы (bash 4+)

```bash
#!/bin/bash

# Группировка в ассоциативный массив
declare -rA DATABASE=(
    [host]="localhost"
    [port]="5432"
    [name]="myapp"
    [user]="appuser"
)

echo "Connecting to ${DATABASE[host]}:${DATABASE[port]}"
```

## Разница между readonly и export

```bash
# readonly - защита от изменения
readonly LOCAL_CONST="value"

# export - передача в дочерние процессы
export EXPORTED_VAR="value"

# Комбинация - защита + передача
export readonly IMPORTANT_CONFIG="value"
# или
declare -rx IMPORTANT_CONFIG="value"
```

## Ограничения readonly

### Нельзя снять readonly

```bash
readonly MY_CONST="value"
unset MY_CONST  # bash: unset: MY_CONST: cannot unset: readonly variable

# Обход: readonly действует только в текущем shell
bash -c 'MY_CONST="new value"; echo $MY_CONST'  # Работает в подпроцессе
```

### Readonly не наследуется

```bash
readonly PARENT_CONST="value"
export PARENT_CONST

# В дочернем процессе переменная есть, но не readonly
bash -c 'echo $PARENT_CONST; PARENT_CONST="changed"; echo $PARENT_CONST'
# value
# changed
```

## Best Practices

### 1. Объявляйте константы в начале скрипта

```bash
#!/bin/bash

# ========== КОНСТАНТЫ ==========
readonly SCRIPT_VERSION="1.0.0"
readonly CONFIG_FILE="/etc/app.conf"
# ================================

# ... остальной код ...
```

### 2. Документируйте константы

```bash
# Максимальное время ожидания ответа от API (секунды)
readonly API_TIMEOUT=30

# Количество попыток переподключения при ошибке
readonly MAX_RETRIES=3

# Интервал между попытками (секунды)
readonly RETRY_INTERVAL=5
```

### 3. Используйте значения по умолчанию разумно

```bash
# Позволяет переопределить через переменную окружения
readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"
readonly MAX_WORKERS="${MAX_WORKERS:-4}"

# Но путь к конфигу лучше зафиксировать
readonly CONFIG_FILE="/etc/app/config.yaml"
```

### 4. Группируйте связанные константы

```bash
#!/bin/bash

# ==========================================
# Пути к директориям
# ==========================================
readonly BASE_DIR="/opt/myapp"
readonly BIN_DIR="${BASE_DIR}/bin"
readonly LIB_DIR="${BASE_DIR}/lib"
readonly LOG_DIR="${BASE_DIR}/logs"
readonly TMP_DIR="${BASE_DIR}/tmp"

# ==========================================
# Ограничения и лимиты
# ==========================================
readonly MAX_FILE_SIZE=104857600  # 100MB в байтах
readonly MAX_UPLOAD_COUNT=10
readonly SESSION_TIMEOUT=3600
```
