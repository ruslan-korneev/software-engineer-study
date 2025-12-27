# Код возврата (Exit Status)

## Что такое код возврата

Каждая команда в Linux возвращает целое число от 0 до 255, называемое **кодом возврата** (exit status, exit code, return code).

- **0** - успешное выполнение
- **1-255** - ошибка (разные значения могут означать разные типы ошибок)

## Переменная $?

Код возврата последней команды хранится в специальной переменной `$?`:

```bash
ls /tmp
echo $?  # 0 - успех

ls /nonexistent
echo $?  # 2 - файл не найден (для ls)
```

### Пример использования

```bash
#!/bin/bash

grep "pattern" file.txt
status=$?

if [[ $status -eq 0 ]]; then
    echo "Паттерн найден"
elif [[ $status -eq 1 ]]; then
    echo "Паттерн не найден"
else
    echo "Произошла ошибка (код: $status)"
fi
```

## Стандартные коды возврата

| Код | Значение |
|-----|----------|
| 0 | Успешное выполнение |
| 1 | Общая ошибка |
| 2 | Неправильное использование команды |
| 126 | Команда найдена, но не исполняемая |
| 127 | Команда не найдена |
| 128 | Недопустимый аргумент для exit |
| 128+N | Программа завершена сигналом N |
| 130 | Завершено по Ctrl+C (SIGINT = 2) |
| 137 | Завершено по SIGKILL (128+9) |
| 143 | Завершено по SIGTERM (128+15) |

### Примеры

```bash
# Команда не найдена
nonexistent_command
echo $?  # 127

# Нет прав на выполнение
chmod -x script.sh
./script.sh
echo $?  # 126

# Ctrl+C во время выполнения
sleep 100  # Нажать Ctrl+C
echo $?  # 130
```

## Команда exit

Команда `exit` завершает скрипт с указанным кодом:

```bash
#!/bin/bash

# Успешное завершение
exit 0

# Завершение с ошибкой
exit 1

# Без аргумента - код последней команды
exit
```

### Использование в функциях

Для функций используется `return` вместо `exit`:

```bash
check_file() {
    local file=$1

    if [[ -f "$file" ]]; then
        return 0
    else
        return 1
    fi
}

# Использование
if check_file "/etc/passwd"; then
    echo "Файл существует"
fi
```

**Важно:**
- `exit` завершает весь скрипт
- `return` завершает только функцию

## if и код возврата

Оператор `if` проверяет код возврата команды:

```bash
# if выполняет команду и проверяет её код возврата
if grep -q "root" /etc/passwd; then
    echo "Пользователь root существует"
fi

# Эквивалентно:
grep -q "root" /etc/passwd
if [[ $? -eq 0 ]]; then
    echo "Пользователь root существует"
fi
```

### Команда как условие

```bash
# Проверка существования команды
if command -v git &>/dev/null; then
    echo "Git установлен"
else
    echo "Git не найден"
fi

# Проверка процесса
if pgrep nginx >/dev/null; then
    echo "Nginx запущен"
fi

# Проверка подключения
if ping -c 1 google.com &>/dev/null; then
    echo "Есть интернет"
fi
```

## Инвертирование кода возврата

### Оператор !

```bash
if ! grep -q "pattern" file.txt; then
    echo "Паттерн НЕ найден"
fi

# Эквивалентно
if grep -q "pattern" file.txt; then
    :  # ничего не делаем
else
    echo "Паттерн НЕ найден"
fi
```

## Коды возврата в цепочках

### Оператор &&

Выполняет следующую команду, только если предыдущая успешна (код 0):

```bash
mkdir new_dir && cd new_dir && touch file.txt
# Если mkdir не удастся, cd и touch не выполнятся
```

### Оператор ||

Выполняет следующую команду, только если предыдущая неуспешна (код != 0):

```bash
cd /nonexistent || echo "Директория не существует"
# echo выполнится, если cd не удастся
```

### Комбинация

```bash
# Паттерн: команда || обработка_ошибки
cd "$dir" || { echo "Ошибка"; exit 1; }

# Паттерн: условие && успех || неудача
[[ -f "$file" ]] && echo "Есть" || echo "Нет"
```

## Код возврата в pipe

В pipe код возврата берётся от последней команды:

```bash
false | true
echo $?  # 0 (от true)

true | false
echo $?  # 1 (от false)
```

### PIPESTATUS

Массив `PIPESTATUS` содержит коды всех команд в pipe:

```bash
false | true | false
echo "${PIPESTATUS[@]}"  # 1 0 1
echo "${PIPESTATUS[0]}"  # 1 (первая команда)
echo "${PIPESTATUS[1]}"  # 0 (вторая команда)
```

### set -o pipefail

С этой опцией pipe возвращает код первой неуспешной команды:

```bash
set -o pipefail

false | true
echo $?  # 1 (от false)
```

## Практические примеры

### Проверка успешности операции

```bash
#!/bin/bash

backup_file="/backup/data.tar.gz"

if tar -czf "$backup_file" /var/data; then
    echo "Резервная копия создана успешно"
else
    echo "Ошибка при создании резервной копии"
    exit 1
fi
```

### Скрипт с кодами ошибок

```bash
#!/bin/bash

# Определение кодов ошибок
readonly E_SUCCESS=0
readonly E_NO_ARGS=1
readonly E_FILE_NOT_FOUND=2
readonly E_PERMISSION_DENIED=3

# Проверка аргументов
if [[ $# -eq 0 ]]; then
    echo "Ошибка: не указан файл" >&2
    exit $E_NO_ARGS
fi

file=$1

# Проверка существования
if [[ ! -e "$file" ]]; then
    echo "Ошибка: файл не найден" >&2
    exit $E_FILE_NOT_FOUND
fi

# Проверка прав
if [[ ! -r "$file" ]]; then
    echo "Ошибка: нет прав на чтение" >&2
    exit $E_PERMISSION_DENIED
fi

# Обработка файла...
echo "Обработка: $file"
exit $E_SUCCESS
```

### Обработка нескольких команд

```bash
#!/bin/bash

errors=0

# Выполняем несколько операций
cp file1.txt /backup/ || ((errors++))
cp file2.txt /backup/ || ((errors++))
cp file3.txt /backup/ || ((errors++))

if [[ $errors -gt 0 ]]; then
    echo "Завершено с $errors ошибками"
    exit 1
else
    echo "Все операции успешны"
    exit 0
fi
```

### Trap для обработки ошибок

```bash
#!/bin/bash
set -e  # Выход при первой ошибке

cleanup() {
    echo "Очистка..."
    rm -f /tmp/tempfile
}

trap cleanup EXIT

# Код скрипта...
echo "Выполнение..."
```
