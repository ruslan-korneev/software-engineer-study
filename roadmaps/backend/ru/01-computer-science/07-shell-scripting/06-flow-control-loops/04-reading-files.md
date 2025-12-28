# Чтение файлов в циклах

[prev: 03-break-continue](./03-break-continue.md) | [next: 01-syntax-errors](../07-debugging/01-syntax-errors.md)

---
## Стандартный паттерн чтения файла

```bash
#!/bin/bash

while IFS= read -r line; do
    echo "Строка: $line"
done < filename.txt
```

### Разбор конструкции

- `IFS=` - предотвращает удаление начальных/конечных пробелов
- `read -r` - отключает обработку escape-последовательностей
- `line` - переменная для строки
- `< filename.txt` - перенаправление файла на вход цикла

## Варианты чтения файлов

### Чтение всего файла

```bash
# Построчное чтение
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Последняя строка без newline (важно!)
while IFS= read -r line || [[ -n "$line" ]]; do
    echo "$line"
done < file.txt
```

### Чтение с сохранением пустых строк

```bash
while IFS= read -r line; do
    if [[ -z "$line" ]]; then
        echo "[пустая строка]"
    else
        echo "$line"
    fi
done < file.txt
```

### Чтение в массив

```bash
# Весь файл в массив (построчно)
mapfile -t lines < file.txt
# или
readarray -t lines < file.txt

# Использование
for line in "${lines[@]}"; do
    echo "$line"
done

echo "Всего строк: ${#lines[@]}"
```

## Чтение CSV и разделённых данных

### Простой CSV

```bash
# data.csv:
# name,age,city
# Alice,25,Moscow

while IFS=',' read -r name age city; do
    echo "Имя: $name, Возраст: $age, Город: $city"
done < data.csv
```

### Пропуск заголовка

```bash
{
    read -r header  # Прочитать первую строку отдельно
    while IFS=',' read -r name age city; do
        echo "$name - $age - $city"
    done
} < data.csv
```

### Обработка /etc/passwd

```bash
while IFS=':' read -r user pass uid gid gecos home shell; do
    if [[ $uid -ge 1000 && "$shell" != "/usr/sbin/nologin" ]]; then
        echo "Пользователь: $user, Домашняя: $home"
    fi
done < /etc/passwd
```

### TSV (табуляция как разделитель)

```bash
while IFS=$'\t' read -r col1 col2 col3; do
    echo "Колонки: $col1 | $col2 | $col3"
done < data.tsv
```

## Чтение из команд

### Process Substitution

```bash
# Рекомендуемый способ - сохраняет переменные
while IFS= read -r file; do
    echo "Найден: $file"
    ((count++))
done < <(find /tmp -type f -name "*.log")

echo "Всего файлов: $count"
```

### Pipe (с ограничениями)

```bash
# Переменные внутри цикла НЕ видны снаружи (subshell)
count=0
find /tmp -type f -name "*.log" | while IFS= read -r file; do
    ((count++))
done
echo "Счётчик: $count"  # 0! (переменная в subshell)

# Обходной путь с lastpipe (bash 4.2+)
shopt -s lastpipe
count=0
find /tmp -name "*.log" | while IFS= read -r file; do
    ((count++))
done
echo "Счётчик: $count"  # Корректное значение
```

### Here String

```bash
data="line1
line2
line3"

while IFS= read -r line; do
    echo ">> $line"
done <<< "$data"
```

## Чтение нескольких файлов

### Последовательное чтение

```bash
for file in *.txt; do
    echo "=== Файл: $file ==="
    while IFS= read -r line; do
        echo "$line"
    done < "$file"
done
```

### Параллельное чтение (сравнение)

```bash
# Чтение из двух файлов одновременно
exec 3< file1.txt
exec 4< file2.txt

while IFS= read -r line1 <&3 && IFS= read -r line2 <&4; do
    if [[ "$line1" != "$line2" ]]; then
        echo "Различие: '$line1' vs '$line2'"
    fi
done

exec 3<&-
exec 4<&-
```

## Чтение с обработкой ошибок

### Проверка существования файла

```bash
input_file="$1"

if [[ ! -f "$input_file" ]]; then
    echo "Файл не найден: $input_file" >&2
    exit 1
fi

if [[ ! -r "$input_file" ]]; then
    echo "Нет прав на чтение: $input_file" >&2
    exit 1
fi

while IFS= read -r line; do
    # обработка
    :
done < "$input_file"
```

### Обработка ошибок в цикле

```bash
while IFS= read -r line; do
    # Пропуск пустых строк
    [[ -z "$line" ]] && continue

    # Пропуск комментариев
    [[ "$line" == \#* ]] && continue

    # Обработка с проверкой ошибок
    if ! process "$line"; then
        echo "Ошибка обработки: $line" >&2
        continue  # или break/exit
    fi
done < input.txt
```

## Практические примеры

### Парсинг конфигурационного файла

```bash
#!/bin/bash

# config.conf:
# # Комментарий
# host=localhost
# port=8080

declare -A config

while IFS='=' read -r key value; do
    # Пропуск пустых строк и комментариев
    [[ -z "$key" || "$key" == \#* ]] && continue

    # Удаление пробелов
    key=$(echo "$key" | xargs)
    value=$(echo "$value" | xargs)

    config["$key"]="$value"
done < config.conf

echo "Host: ${config[host]}"
echo "Port: ${config[port]}"
```

### Обработка лог-файла

```bash
#!/bin/bash

# Подсчёт ошибок по типу
declare -A error_counts

while IFS= read -r line; do
    if [[ "$line" == *"ERROR"* ]]; then
        # Извлечение типа ошибки (например, после ERROR:)
        if [[ "$line" =~ ERROR:\ ([^:]+) ]]; then
            error_type="${BASH_REMATCH[1]}"
            ((error_counts["$error_type"]++))
        fi
    fi
done < /var/log/app.log

echo "Статистика ошибок:"
for type in "${!error_counts[@]}"; do
    echo "  $type: ${error_counts[$type]}"
done
```

### Массовое переименование из списка

```bash
#!/bin/bash

# rename_list.txt:
# old_name1.txt new_name1.txt
# old_name2.txt new_name2.txt

while read -r old_name new_name; do
    if [[ -f "$old_name" ]]; then
        mv "$old_name" "$new_name"
        echo "Переименовано: $old_name -> $new_name"
    else
        echo "Не найден: $old_name" >&2
    fi
done < rename_list.txt
```

### Загрузка URL из файла

```bash
#!/bin/bash

# urls.txt:
# https://example.com/file1.txt
# https://example.com/file2.txt

while IFS= read -r url; do
    [[ -z "$url" || "$url" == \#* ]] && continue

    filename=$(basename "$url")
    echo "Загрузка: $url"

    if curl -sO "$url"; then
        echo "OK: $filename"
    else
        echo "ОШИБКА: $filename" >&2
    fi
done < urls.txt
```

### Чтение и запись

```bash
#!/bin/bash

# Обработка файла с сохранением результата
while IFS= read -r line; do
    # Какая-то трансформация
    echo "${line^^}"  # Преобразование в верхний регистр
done < input.txt > output.txt

# Или добавление
while IFS= read -r line; do
    echo "[$(date)] $line"
done < input.txt >> log.txt
```

## Best Practices

1. **Всегда используйте `IFS= read -r`** для сохранения форматирования
2. **Обрабатывайте последнюю строку** без newline: `|| [[ -n "$line" ]]`
3. **Проверяйте существование файла** перед чтением
4. **Используйте process substitution** вместо pipe для сохранения переменных
5. **Закрывайте файловые дескрипторы** при использовании exec

---

[prev: 03-break-continue](./03-break-continue.md) | [next: 01-syntax-errors](../07-debugging/01-syntax-errors.md)
