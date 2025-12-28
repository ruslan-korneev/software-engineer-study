# Цикл until

[prev: 01-while-loop](./01-while-loop.md) | [next: 03-break-continue](./03-break-continue.md)

---
## Базовый синтаксис

Цикл `until` выполняет команды, пока условие **ложно** (противоположность `while`):

```bash
until условие; do
    команды
done
```

## Сравнение while и until

```bash
# while: пока условие ИСТИННО
while [[ $count -lt 5 ]]; do
    echo $count
    ((count++))
done

# until: пока условие ЛОЖНО (до тех пор, пока не станет истинным)
until [[ $count -ge 5 ]]; do
    echo $count
    ((count++))
done
```

Оба цикла выдают одинаковый результат, но логика обратная.

## Простой пример

```bash
#!/bin/bash

count=1

until [[ $count -gt 5 ]]; do
    echo "Счётчик: $count"
    ((count++))
done
```

Вывод:
```
Счётчик: 1
Счётчик: 2
Счётчик: 3
Счётчик: 4
Счётчик: 5
```

## Когда использовать until

`until` более естественен, когда вы **ждёте наступления события**:

### Ожидание появления файла

```bash
#!/bin/bash

echo "Ожидание файла /tmp/ready.txt..."

until [[ -f "/tmp/ready.txt" ]]; do
    echo "Файл не найден, ожидание..."
    sleep 2
done

echo "Файл появился!"
```

### Ожидание доступности сервиса

```bash
#!/bin/bash

host="database.example.com"
port=5432

echo "Ожидание доступности $host:$port..."

until nc -z "$host" "$port" 2>/dev/null; do
    echo "Сервис недоступен, повторная попытка..."
    sleep 3
done

echo "Сервис доступен!"
```

### Ожидание завершения процесса

```bash
#!/bin/bash

pid=$1
echo "Ожидание завершения процесса $pid..."

until ! kill -0 "$pid" 2>/dev/null; do
    echo "Процесс всё ещё работает..."
    sleep 1
done

echo "Процесс завершился"
```

## Практические примеры

### Обратный отсчёт

```bash
#!/bin/bash

seconds=10

until [[ $seconds -eq 0 ]]; do
    echo "Осталось: $seconds"
    ((seconds--))
    sleep 1
done

echo "Время вышло!"
```

### Ввод до получения корректного значения

```bash
#!/bin/bash

valid_input=false

until $valid_input; do
    read -rp "Введите число от 1 до 10: " number

    if [[ "$number" =~ ^[0-9]+$ ]] && [[ $number -ge 1 && $number -le 10 ]]; then
        valid_input=true
    else
        echo "Некорректный ввод. Попробуйте снова."
    fi
done

echo "Вы ввели: $number"
```

### Попытки подключения

```bash
#!/bin/bash

max_attempts=10
attempt=1
connected=false

until $connected || [[ $attempt -gt $max_attempts ]]; do
    echo "Попытка $attempt из $max_attempts..."

    if ping -c 1 -W 2 server.example.com &>/dev/null; then
        connected=true
    else
        ((attempt++))
        sleep 2
    fi
done

if $connected; then
    echo "Подключение успешно!"
else
    echo "Не удалось подключиться после $max_attempts попыток"
    exit 1
fi
```

### Ожидание освобождения ресурса

```bash
#!/bin/bash

lockfile="/var/lock/myapp.lock"

until mkdir "$lockfile" 2>/dev/null; do
    echo "Ресурс заблокирован, ожидание..."
    sleep 1
done

echo "Блокировка получена"
# ... работа ...
rmdir "$lockfile"
```

### Мониторинг лог-файла

```bash
#!/bin/bash

logfile="/var/log/app.log"
pattern="ERROR"

# Ожидание появления ошибки в логе
until grep -q "$pattern" "$logfile"; do
    echo "Ошибок пока нет..."
    sleep 10
done

echo "Обнаружена ошибка в логе!"
grep "$pattern" "$logfile" | tail -5
```

## until с командами как условием

```bash
#!/bin/bash

# Ожидание успешного выполнения команды
until make build 2>/dev/null; do
    echo "Сборка не удалась, исправление..."
    # ... какие-то действия ...
    sleep 5
done

echo "Сборка успешна!"
```

## Вложенные until

```bash
#!/bin/bash

outer=1

until [[ $outer -gt 3 ]]; do
    inner=1

    until [[ $inner -gt 3 ]]; do
        echo "outer=$outer, inner=$inner"
        ((inner++))
    done

    ((outer++))
done
```

## Преобразование while в until

Любой `while` можно преобразовать в `until` и наоборот, инвертировав условие:

```bash
# while версия
while [[ $x -lt 10 ]]; do
    ((x++))
done

# Эквивалентный until (инвертируем условие)
until [[ $x -ge 10 ]]; do
    ((x++))
done

# while с NOT
while [[ ! -f "$file" ]]; do
    sleep 1
done

# Эквивалентный until (убираем NOT)
until [[ -f "$file" ]]; do
    sleep 1
done
```

## break и continue в until

```bash
#!/bin/bash

count=0

until false; do  # Бесконечный цикл
    ((count++))

    if [[ $count -eq 3 ]]; then
        echo "Пропускаем 3"
        continue
    fi

    if [[ $count -gt 5 ]]; then
        echo "Достигли предела"
        break
    fi

    echo "Счётчик: $count"
done
```

## Выбор между while и until

| Ситуация | Рекомендация |
|----------|--------------|
| Выполнять пока условие истинно | `while` |
| Выполнять пока условие ложно | `until` |
| Ожидание события | `until` (более читабельно) |
| Обычный счётчик | `while` (более привычно) |
| Чтение файла | `while` |

### Примеры выбора

```bash
# while - более естественно для счётчика
i=0
while [[ $i -lt 10 ]]; do
    ((i++))
done

# until - более естественно для ожидания
until [[ -f "$ready_file" ]]; do
    sleep 1
done

# until - более естественно для "до тех пор, пока не..."
until ping -c 1 server.com &>/dev/null; do
    echo "Сервер недоступен"
    sleep 5
done
```

---

[prev: 01-while-loop](./01-while-loop.md) | [next: 03-break-continue](./03-break-continue.md)
