# Оператор case

[prev: 04-tracing](../07-debugging/04-tracing.md) | [next: 02-patterns](./02-patterns.md)

---
## Базовый синтаксис

Оператор `case` позволяет выбирать между несколькими вариантами на основе значения переменной:

```bash
case выражение in
    паттерн1)
        команды
        ;;
    паттерн2)
        команды
        ;;
    *)
        команды по умолчанию
        ;;
esac
```

### Простой пример

```bash
#!/bin/bash

read -rp "Введите букву (a, b или c): " letter

case $letter in
    a)
        echo "Вы ввели A"
        ;;
    b)
        echo "Вы ввели B"
        ;;
    c)
        echo "Вы ввели C"
        ;;
    *)
        echo "Неизвестная буква"
        ;;
esac
```

## Сравнение case и if-elif

### Эквивалентный код

```bash
# С case - более читабельно
case $choice in
    1) echo "Один" ;;
    2) echo "Два" ;;
    3) echo "Три" ;;
    *) echo "Другое" ;;
esac

# С if-elif - менее читабельно для множества вариантов
if [[ $choice == "1" ]]; then
    echo "Один"
elif [[ $choice == "2" ]]; then
    echo "Два"
elif [[ $choice == "3" ]]; then
    echo "Три"
else
    echo "Другое"
fi
```

### Когда использовать case

- Множество дискретных значений для проверки
- Сопоставление с паттернами
- Обработка опций командной строки
- Меню

### Когда использовать if

- Сложные условия
- Диапазоны чисел
- Комбинации условий с && и ||

## Несколько команд в case

```bash
#!/bin/bash

case $1 in
    start)
        echo "Запуск сервиса..."
        systemctl start myservice
        echo "Сервис запущен"
        ;;
    stop)
        echo "Остановка сервиса..."
        systemctl stop myservice
        echo "Сервис остановлен"
        ;;
esac
```

## Множественные паттерны

Можно объединять несколько паттернов через `|`:

```bash
#!/bin/bash

read -rp "Продолжить? (y/n): " answer

case $answer in
    y|Y|yes|Yes|YES|да|Да|ДА)
        echo "Продолжаем..."
        ;;
    n|N|no|No|NO|нет|Нет|НЕТ)
        echo "Отмена"
        ;;
    *)
        echo "Неизвестный ответ"
        ;;
esac
```

## Паттерн по умолчанию

Паттерн `*` соответствует всему остальному:

```bash
case $command in
    help)
        show_help
        ;;
    version)
        show_version
        ;;
    *)
        echo "Неизвестная команда: $command"
        echo "Используйте 'help' для справки"
        exit 1
        ;;
esac
```

## Fallthrough (;&) и Continue (;;&)

### ;; - завершение (по умолчанию)

```bash
case $x in
    1) echo "Один" ;;  # После выполнения выход из case
    2) echo "Два" ;;
esac
```

### ;& - fallthrough (bash 4.0+)

Продолжить выполнение следующего блока:

```bash
case $level in
    debug)
        echo "DEBUG"
        ;&
    info)
        echo "INFO"
        ;&
    warning)
        echo "WARNING"
        ;&
    error)
        echo "ERROR"
        ;;
esac

# При level=debug выведет: DEBUG INFO WARNING ERROR
# При level=warning выведет: WARNING ERROR
```

### ;;& - continue (bash 4.0+)

Проверить следующий паттерн:

```bash
case $str in
    *a*)
        echo "Содержит 'a'"
        ;;&
    *b*)
        echo "Содержит 'b'"
        ;;&
    *c*)
        echo "Содержит 'c'"
        ;;
esac

# При str="abc" выведет все три строки
# При str="ab" выведет первые две
```

## Практические примеры

### Обработка аргументов командной строки

```bash
#!/bin/bash

case $1 in
    -h|--help)
        echo "Использование: $0 [опции]"
        echo "  -h, --help     Показать справку"
        echo "  -v, --version  Показать версию"
        exit 0
        ;;
    -v|--version)
        echo "Версия 1.0.0"
        exit 0
        ;;
    "")
        echo "Не указаны аргументы"
        exit 1
        ;;
    *)
        echo "Неизвестная опция: $1"
        exit 1
        ;;
esac
```

### Init-скрипт для сервиса

```bash
#!/bin/bash

SERVICE_NAME="myapp"

case $1 in
    start)
        echo "Starting $SERVICE_NAME..."
        /usr/bin/myapp &
        echo $! > /var/run/myapp.pid
        ;;
    stop)
        echo "Stopping $SERVICE_NAME..."
        if [[ -f /var/run/myapp.pid ]]; then
            kill $(cat /var/run/myapp.pid)
            rm /var/run/myapp.pid
        fi
        ;;
    restart)
        $0 stop
        sleep 2
        $0 start
        ;;
    status)
        if [[ -f /var/run/myapp.pid ]]; then
            echo "$SERVICE_NAME is running (PID: $(cat /var/run/myapp.pid))"
        else
            echo "$SERVICE_NAME is not running"
        fi
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

### Определение типа файла

```bash
#!/bin/bash

file=$1

case $file in
    *.txt)
        echo "Текстовый файл"
        cat "$file"
        ;;
    *.jpg|*.jpeg|*.png|*.gif)
        echo "Изображение"
        file "$file"
        ;;
    *.sh)
        echo "Shell-скрипт"
        bash -n "$file" && echo "Синтаксис OK"
        ;;
    *.tar.gz|*.tgz)
        echo "Архив tar.gz"
        tar -tzf "$file"
        ;;
    *)
        echo "Неизвестный тип файла"
        file "$file"
        ;;
esac
```

### Интерактивное меню

```bash
#!/bin/bash

while true; do
    echo ""
    echo "=== Главное меню ==="
    echo "1. Показать дату"
    echo "2. Показать диск"
    echo "3. Показать память"
    echo "q. Выход"
    echo ""
    read -rp "Выберите опцию: " choice

    case $choice in
        1)
            date
            ;;
        2)
            df -h
            ;;
        3)
            free -h
            ;;
        q|Q)
            echo "До свидания!"
            break
            ;;
        *)
            echo "Неверный выбор"
            ;;
    esac

    read -rp "Нажмите Enter..."
done
```

### Определение операционной системы

```bash
#!/bin/bash

case "$OSTYPE" in
    linux*)
        echo "Linux"
        PACKAGE_MGR="apt"
        ;;
    darwin*)
        echo "macOS"
        PACKAGE_MGR="brew"
        ;;
    cygwin*|msys*|mingw*)
        echo "Windows"
        PACKAGE_MGR="choco"
        ;;
    *)
        echo "Неизвестная ОС: $OSTYPE"
        ;;
esac
```

## Case в функциях

```bash
#!/bin/bash

get_file_type() {
    case $1 in
        *.txt)          echo "text" ;;
        *.jpg|*.png)    echo "image" ;;
        *.mp3|*.wav)    echo "audio" ;;
        *.mp4|*.avi)    echo "video" ;;
        *)              echo "unknown" ;;
    esac
}

# Использование
file_type=$(get_file_type "photo.jpg")
echo "Тип файла: $file_type"
```

## Вложенный case

```bash
#!/bin/bash

case $1 in
    config)
        case $2 in
            set)
                echo "Установка: $3=$4"
                ;;
            get)
                echo "Получение: $3"
                ;;
            *)
                echo "Usage: $0 config {set|get}"
                ;;
        esac
        ;;
    *)
        echo "Usage: $0 {config}"
        ;;
esac
```

---

[prev: 04-tracing](../07-debugging/04-tracing.md) | [next: 02-patterns](./02-patterns.md)
