# Именованные каналы (Named Pipes / FIFO)

## Что такое именованные каналы

**Именованный канал (FIFO)** - это специальный файл, который работает как pipe, но существует в файловой системе. Позволяет связать два независимых процесса.

FIFO = First In, First Out (первым вошёл - первым вышел)

## Создание FIFO

```bash
# Через mkfifo
mkfifo /tmp/myfifo

# С указанием прав
mkfifo -m 644 /tmp/myfifo

# Проверка типа файла
ls -l /tmp/myfifo
# prw-r--r-- 1 user group 0 ... /tmp/myfifo
# ^ 'p' означает pipe
```

## Базовое использование

### Терминал 1 (читатель)

```bash
cat /tmp/myfifo  # Блокируется до появления данных
```

### Терминал 2 (писатель)

```bash
echo "Hello, FIFO!" > /tmp/myfifo
```

После записи в терминале 1 появится "Hello, FIFO!" и cat завершится.

## Особенности поведения

### Блокировка при открытии

- **Чтение** из FIFO блокируется, пока кто-то не откроет его для записи
- **Запись** в FIFO блокируется, пока кто-то не откроет его для чтения

```bash
# Это заблокируется:
cat /tmp/myfifo  # Ждёт писателя

# Это тоже заблокируется:
echo "data" > /tmp/myfifo  # Ждёт читателя
```

### Избежание блокировки

```bash
# Открыть для чтения и записи (не блокирует)
exec 3<>/tmp/myfifo

# Или запись в фоне
echo "data" > /tmp/myfifo &
cat /tmp/myfifo
```

## Практические примеры

### Простой чат

#### Терминал 1

```bash
#!/bin/bash
mkfifo /tmp/chat_pipe 2>/dev/null

echo "Ожидание сообщений..."
while true; do
    if read line < /tmp/chat_pipe; then
        echo "Получено: $line"
    fi
done
```

#### Терминал 2

```bash
#!/bin/bash
while true; do
    read -p "Сообщение: " msg
    echo "$msg" > /tmp/chat_pipe
done
```

### Логирование

```bash
#!/bin/bash

LOGPIPE="/tmp/log_pipe"
LOGFILE="/var/log/myapp.log"

# Создаём FIFO
mkfifo "$LOGPIPE" 2>/dev/null

# Запускаем логгер в фоне
while true; do
    if read -r line < "$LOGPIPE"; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] $line" >> "$LOGFILE"
    fi
done &

logger_pid=$!

# Функция для логирования
log() {
    echo "$*" > "$LOGPIPE"
}

# Использование
log "Приложение запущено"
log "Обработка данных..."
log "Завершено"

# Очистка
kill $logger_pid 2>/dev/null
rm -f "$LOGPIPE"
```

### Межпроцессное взаимодействие

#### Сервер

```bash
#!/bin/bash

REQUEST_PIPE="/tmp/request"
RESPONSE_PIPE="/tmp/response"

mkfifo "$REQUEST_PIPE" "$RESPONSE_PIPE" 2>/dev/null

echo "Сервер запущен"

while true; do
    # Читаем запрос
    if read -r request < "$REQUEST_PIPE"; then
        echo "Получен запрос: $request"

        # Обработка и ответ
        case "$request" in
            "time")
                date > "$RESPONSE_PIPE"
                ;;
            "uptime")
                uptime > "$RESPONSE_PIPE"
                ;;
            "quit")
                echo "Завершение" > "$RESPONSE_PIPE"
                break
                ;;
            *)
                echo "Неизвестная команда" > "$RESPONSE_PIPE"
                ;;
        esac
    fi
done

rm -f "$REQUEST_PIPE" "$RESPONSE_PIPE"
```

#### Клиент

```bash
#!/bin/bash

REQUEST_PIPE="/tmp/request"
RESPONSE_PIPE="/tmp/response"

# Отправка запроса
echo "$1" > "$REQUEST_PIPE"

# Получение ответа
cat "$RESPONSE_PIPE"
```

### Producer-Consumer

```bash
#!/bin/bash

PIPE="/tmp/work_queue"
mkfifo "$PIPE" 2>/dev/null

# Producer
producer() {
    for i in {1..10}; do
        echo "Task $i" > "$PIPE"
        sleep 0.5
    done
    echo "DONE" > "$PIPE"
}

# Consumer
consumer() {
    while true; do
        read -r task < "$PIPE"
        if [[ "$task" == "DONE" ]]; then
            break
        fi
        echo "Обработка: $task"
        sleep 1
    done
}

# Запуск
producer &
consumer

rm -f "$PIPE"
```

### Прогресс-бар через FIFO

```bash
#!/bin/bash

PROGRESS_PIPE="/tmp/progress"
mkfifo "$PROGRESS_PIPE" 2>/dev/null

# Отображение прогресса
show_progress() {
    while read -r percent < "$PROGRESS_PIPE"; do
        [[ "$percent" == "DONE" ]] && break
        printf "\rПрогресс: %3d%%" "$percent"
    done
    echo ""
}

# Симуляция работы
do_work() {
    for i in {1..100}; do
        sleep 0.05
        echo "$i" > "$PROGRESS_PIPE"
    done
    echo "DONE" > "$PROGRESS_PIPE"
}

show_progress &
do_work

rm -f "$PROGRESS_PIPE"
```

## Сравнение с обычными pipe

| Характеристика | Обычный pipe (`|`) | Именованный pipe (FIFO) |
|----------------|-------------------|------------------------|
| Существование | Только во время выполнения | Постоянный файл |
| Связь процессов | Родитель-потомок | Любые процессы |
| Создание | Автоматическое | Явное (`mkfifo`) |
| Двунаправленность | Нет | Нет (но можно два FIFO) |
| Удаление | Автоматическое | Ручное (`rm`) |

## Best Practices

1. **Удаляйте FIFO** после использования
2. **Используйте trap** для очистки
3. **Обрабатывайте блокировки** (таймауты, фоновые процессы)
4. **Проверяйте существование** перед созданием

```bash
#!/bin/bash

FIFO="/tmp/myfifo"

cleanup() {
    rm -f "$FIFO"
}

trap cleanup EXIT

# Создаём только если не существует
[[ -p "$FIFO" ]] || mkfifo "$FIFO"

# Использование...
```

## Ограничения

- Однонаправленные (для двусторонней связи нужны два FIFO)
- Блокируют при открытии без партнёра
- Размер буфера ограничен (обычно 64KB)
- Данные читаются только один раз
