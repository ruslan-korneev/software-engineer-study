# Цикл while

[prev: 04-menus](../05-reading-input/04-menus.md) | [next: 02-until-loop](./02-until-loop.md)

---
## Базовый синтаксис

Цикл `while` выполняет команды, пока условие истинно:

```bash
while условие; do
    команды
done
```

### Простой пример

```bash
#!/bin/bash

count=1

while [[ $count -le 5 ]]; do
    echo "Итерация: $count"
    ((count++))
done
```

Вывод:
```
Итерация: 1
Итерация: 2
Итерация: 3
Итерация: 4
Итерация: 5
```

## Условие цикла

### С [[ ]] или [ ]

```bash
# Числовое сравнение
while [[ $x -lt 10 ]]; do
    echo $x
    ((x++))
done

# Строковое сравнение
while [[ "$answer" != "quit" ]]; do
    read -rp "Введите команду: " answer
done
```

### С (( )) для чисел

```bash
i=0
while (( i < 10 )); do
    echo $i
    ((i++))
done
```

### С командой как условием

```bash
# Цикл пока команда успешна
while ping -c 1 -W 1 server.example.com &>/dev/null; do
    echo "Сервер доступен"
    sleep 5
done
echo "Сервер недоступен!"

# Чтение файла построчно
while IFS= read -r line; do
    echo "Строка: $line"
done < file.txt
```

## Бесконечный цикл

```bash
# Способ 1: с true
while true; do
    echo "Работаю..."
    sleep 1
done

# Способ 2: с :
while :; do
    echo "Работаю..."
    sleep 1
done

# Способ 3: с 1
while [[ 1 ]]; do
    echo "Работаю..."
    sleep 1
done
```

### Выход из бесконечного цикла

```bash
while true; do
    read -rp "Введите 'exit' для выхода: " input
    if [[ "$input" == "exit" ]]; then
        break
    fi
    echo "Вы ввели: $input"
done
```

## Чтение файла построчно

### Стандартный паттерн

```bash
#!/bin/bash

while IFS= read -r line; do
    echo "Обработка: $line"
done < input.txt
```

### Чтение с номером строки

```bash
line_number=0

while IFS= read -r line; do
    ((line_number++))
    echo "$line_number: $line"
done < input.txt
```

### Чтение из переменной

```bash
data="line1
line2
line3"

while IFS= read -r line; do
    echo ">> $line"
done <<< "$data"
```

### Чтение из команды

```bash
# Process substitution
while IFS= read -r line; do
    echo "Файл: $line"
done < <(find /tmp -type f -name "*.log")

# Pipe (осторожно с переменными!)
find /tmp -type f -name "*.log" | while IFS= read -r line; do
    echo "Файл: $line"
done
```

## Практические примеры

### Ожидание события

```bash
#!/bin/bash

# Ожидание появления файла
while [[ ! -f "/tmp/signal.txt" ]]; do
    echo "Ожидание файла..."
    sleep 2
done
echo "Файл появился!"
```

### Обратный отсчёт

```bash
#!/bin/bash

countdown=10

while [[ $countdown -gt 0 ]]; do
    echo -ne "\rОсталось: $countdown секунд "
    ((countdown--))
    sleep 1
done
echo -e "\rВремя вышло!          "
```

### Повторные попытки

```bash
#!/bin/bash

max_attempts=5
attempt=1

while [[ $attempt -le $max_attempts ]]; do
    echo "Попытка $attempt из $max_attempts"

    if curl -s http://example.com > /dev/null; then
        echo "Успех!"
        break
    fi

    echo "Неудача, повтор через 5 секунд..."
    sleep 5
    ((attempt++))
done

if [[ $attempt -gt $max_attempts ]]; then
    echo "Все попытки исчерпаны"
    exit 1
fi
```

### Обработка очереди

```bash
#!/bin/bash

# Обработка файлов в очереди
queue_dir="/var/spool/myqueue"

while true; do
    # Найти первый файл в очереди
    file=$(find "$queue_dir" -type f -print -quit)

    if [[ -z "$file" ]]; then
        echo "Очередь пуста, ожидание..."
        sleep 5
        continue
    fi

    echo "Обработка: $file"
    # ... обработка ...
    rm "$file"
done
```

### Мониторинг процесса

```bash
#!/bin/bash

pid=$1

if [[ -z "$pid" ]]; then
    echo "Использование: $0 <PID>"
    exit 1
fi

echo "Мониторинг процесса $pid"

while kill -0 "$pid" 2>/dev/null; do
    echo "Процесс $pid работает..."
    sleep 2
done

echo "Процесс $pid завершился"
```

### Интерактивный ввод

```bash
#!/bin/bash

sum=0

while true; do
    read -rp "Введите число (или 'q' для выхода): " input

    if [[ "$input" == "q" ]]; then
        break
    fi

    if [[ "$input" =~ ^-?[0-9]+$ ]]; then
        ((sum += input))
        echo "Текущая сумма: $sum"
    else
        echo "Это не число!"
    fi
done

echo "Итоговая сумма: $sum"
```

## Вложенные циклы

```bash
#!/bin/bash

# Таблица умножения
i=1
while [[ $i -le 5 ]]; do
    j=1
    while [[ $j -le 5 ]]; do
        printf "%4d" $((i * j))
        ((j++))
    done
    echo
    ((i++))
done
```

## while с несколькими командами

```bash
# Несколько команд в условии (выполняется последняя)
while echo "Проверка..." && [[ $count -lt 5 ]]; do
    echo "count = $count"
    ((count++))
done

# Несколько команд перед проверкой
while
    echo "Итерация"
    date
    [[ $count -lt 3 ]]
do
    ((count++))
done
```

## Изменение IFS в while

```bash
# Чтение CSV
while IFS=',' read -r name age city; do
    echo "Имя: $name, Возраст: $age, Город: $city"
done < data.csv

# Чтение /etc/passwd
while IFS=':' read -r user pass uid gid gecos home shell; do
    echo "User: $user, Shell: $shell"
done < /etc/passwd
```

## Распространённые ошибки

### Проблема с pipe и переменными

```bash
# НЕПРАВИЛЬНО - переменная не видна после цикла
count=0
cat file.txt | while read line; do
    ((count++))
done
echo "Строк: $count"  # 0!

# ПРАВИЛЬНО - использовать перенаправление
count=0
while read line; do
    ((count++))
done < file.txt
echo "Строк: $count"  # Корректное значение
```

### Бесконечный цикл без изменения условия

```bash
# НЕПРАВИЛЬНО - забыли инкремент
i=0
while [[ $i -lt 5 ]]; do
    echo $i
    # ((i++))  # Забыли!
done

# ПРАВИЛЬНО
i=0
while [[ $i -lt 5 ]]; do
    echo $i
    ((i++))
done
```

---

[prev: 04-menus](../05-reading-input/04-menus.md) | [next: 02-until-loop](./02-until-loop.md)
