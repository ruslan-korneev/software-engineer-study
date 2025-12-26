# Обработка сигналов (trap)

## Что такое trap

Команда `trap` позволяет перехватывать сигналы и выполнять код при их получении. Используется для:
- Очистки временных файлов
- Корректного завершения программы
- Обработки ошибок
- Игнорирования сигналов

## Базовый синтаксис

```bash
trap 'команды' СИГНАЛЫ
```

## Основные сигналы

| Сигнал | Номер | Описание |
|--------|-------|----------|
| `EXIT` | - | Выход из скрипта (любым способом) |
| `ERR` | - | Ошибка команды (код != 0) |
| `DEBUG` | - | Перед каждой командой |
| `RETURN` | - | При возврате из функции/source |
| `SIGINT` | 2 | Прерывание (Ctrl+C) |
| `SIGTERM` | 15 | Запрос завершения |
| `SIGHUP` | 1 | Отключение терминала |
| `SIGKILL` | 9 | Принудительное завершение (нельзя перехватить) |

## Примеры использования

### EXIT - очистка при выходе

```bash
#!/bin/bash

# Создаём временный файл
tmp_file=$(mktemp)
echo "Временный файл: $tmp_file"

# Регистрируем очистку
trap 'rm -f "$tmp_file"; echo "Очистка выполнена"' EXIT

# Основная логика
echo "Работаем с файлом..."
sleep 2

# При выходе (любом!) tmp_file будет удалён
```

### SIGINT - обработка Ctrl+C

```bash
#!/bin/bash

trap 'echo ""; echo "Прервано пользователем"; exit 1' SIGINT

echo "Нажмите Ctrl+C для прерывания"
while true; do
    echo "Работаю..."
    sleep 1
done
```

### ERR - обработка ошибок

```bash
#!/bin/bash
set -e  # Выход при ошибке

trap 'echo "Ошибка в строке $LINENO: $BASH_COMMAND"' ERR

echo "Начало"
false  # Команда с ошибкой
echo "Этого не будет"
```

### DEBUG - трассировка

```bash
#!/bin/bash

trap 'echo "DEBUG: $BASH_COMMAND"' DEBUG

x=1
y=2
z=$((x + y))
echo "z = $z"
```

## Множественные сигналы

```bash
#!/bin/bash

cleanup() {
    echo "Очистка..."
    rm -f /tmp/myapp_*
}

trap cleanup EXIT SIGINT SIGTERM SIGHUP
```

## Сброс trap

```bash
#!/bin/bash

# Установка trap
trap 'echo "Caught!"' SIGINT

# Сброс (восстановление поведения по умолчанию)
trap - SIGINT

# Или игнорирование сигнала
trap '' SIGINT  # Ctrl+C ничего не делает
```

## Практические примеры

### Блокировка с очисткой

```bash
#!/bin/bash

LOCKFILE="/var/lock/myscript.lock"

cleanup() {
    rm -f "$LOCKFILE"
    echo "Блокировка снята"
}

# Регистрируем очистку
trap cleanup EXIT

# Пытаемся получить блокировку
if ! mkdir "$LOCKFILE" 2>/dev/null; then
    echo "Скрипт уже запущен"
    exit 1
fi

echo "Блокировка получена"
# Основная работа
sleep 10
```

### Безопасная работа с временными файлами

```bash
#!/bin/bash

# Создаём временную директорию
WORK_DIR=$(mktemp -d)

cleanup() {
    if [[ -d "$WORK_DIR" ]]; then
        rm -rf "$WORK_DIR"
        echo "Временные файлы удалены"
    fi
}

trap cleanup EXIT

# Работаем с временными файлами
cd "$WORK_DIR"
echo "data" > file1.txt
echo "more data" > file2.txt

# Даже при ошибке директория будет удалена
```

### Грациозное завершение сервиса

```bash
#!/bin/bash

shutdown_requested=false

handle_shutdown() {
    echo "Получен сигнал завершения"
    shutdown_requested=true
}

trap handle_shutdown SIGTERM SIGINT

echo "Сервис запущен (PID: $$)"

while ! $shutdown_requested; do
    echo "Обработка..."
    sleep 1
done

echo "Завершение работы..."
# Финализация
sleep 2
echo "Сервис остановлен"
```

### Подтверждение выхода

```bash
#!/bin/bash

confirm_exit() {
    read -p "Вы уверены, что хотите выйти? [y/N] " answer
    if [[ "$answer" == "y" || "$answer" == "Y" ]]; then
        echo "Выход..."
        exit 0
    fi
    echo "Продолжаем работу"
}

trap confirm_exit SIGINT

echo "Нажмите Ctrl+C для выхода"
while true; do
    sleep 1
done
```

### Логирование ошибок

```bash
#!/bin/bash
set -e

LOG_FILE="/var/log/myscript.log"

log_error() {
    local line=$1
    local command=$2
    local code=$3

    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR at line $line" >> "$LOG_FILE"
    echo "  Command: $command" >> "$LOG_FILE"
    echo "  Exit code: $code" >> "$LOG_FILE"
}

trap 'log_error $LINENO "$BASH_COMMAND" $?' ERR

# Код скрипта
echo "Starting..."
false  # Вызовет ошибку
echo "Never reached"
```

### Восстановление настроек терминала

```bash
#!/bin/bash

# Сохраняем настройки терминала
original_tty=$(stty -g)

restore_terminal() {
    stty "$original_tty"
    echo ""
    echo "Терминал восстановлен"
}

trap restore_terminal EXIT

# Изменяем настройки терминала
stty raw -echo

echo "Нажмите любую клавишу..."
read -n 1 key
echo "Вы нажали: $key"
```

## Trap в функциях

### RETURN trap

```bash
#!/bin/bash

my_function() {
    trap 'echo "Возврат из функции"' RETURN
    echo "В функции"
}

my_function
echo "После функции"
```

### Локальный trap

```bash
#!/bin/bash

# trap в функции влияет на весь скрипт
global_effect() {
    trap 'echo "Global trap"' EXIT
}

# Для локального эффекта используйте subshell
local_effect() {
    (
        trap 'echo "Local trap"' EXIT
        echo "В функции"
    )
}
```

## Best Practices

1. **Всегда очищайте временные файлы** через EXIT trap
2. **Используйте функции** для сложной логики очистки
3. **Документируйте** какие сигналы обрабатываются
4. **Тестируйте** обработку сигналов (kill -TERM, Ctrl+C)
5. **Не игнорируйте** SIGTERM без причины
