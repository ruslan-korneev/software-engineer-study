# Традиционный цикл for

## Базовый синтаксис

```bash
for variable in list; do
    commands
done
```

## Итерация по списку значений

### Явный список

```bash
#!/bin/bash

for color in red green blue; do
    echo "Цвет: $color"
done
```

Вывод:
```
Цвет: red
Цвет: green
Цвет: blue
```

### Список с пробелами в значениях

```bash
#!/bin/bash

# Кавычки сохраняют пробелы
for name in "John Doe" "Jane Smith" "Bob Wilson"; do
    echo "Имя: $name"
done
```

## Итерация по последовательности чисел

### Brace expansion

```bash
#!/bin/bash

# От 1 до 5
for i in {1..5}; do
    echo "Число: $i"
done

# От 10 до 1 (обратный порядок)
for i in {10..1}; do
    echo "Обратный отсчёт: $i"
done

# С шагом 2
for i in {0..10..2}; do
    echo "Чётное: $i"
done
```

### Команда seq

```bash
#!/bin/bash

# seq start end
for i in $(seq 1 5); do
    echo $i
done

# seq start step end
for i in $(seq 0 2 10); do
    echo $i
done

# seq с ведущими нулями
for i in $(seq -w 1 10); do
    echo "file_$i.txt"  # file_01.txt, file_02.txt, ...
done
```

## Итерация по файлам

### Glob patterns

```bash
#!/bin/bash

# Все .txt файлы
for file in *.txt; do
    echo "Обработка: $file"
done

# Все файлы в директории
for file in /var/log/*; do
    echo "Файл: $file"
done

# Рекурсивный поиск (bash 4.0+)
shopt -s globstar
for file in **/*.sh; do
    echo "Скрипт: $file"
done
```

### Безопасная обработка файлов

```bash
#!/bin/bash

# Проверка, что файлы существуют
for file in *.txt; do
    # Если нет .txt файлов, glob вернёт "*.txt"
    [[ -e "$file" ]] || continue
    echo "Файл: $file"
done

# Или включить nullglob
shopt -s nullglob
for file in *.txt; do
    echo "Файл: $file"  # Не выполнится, если нет файлов
done
```

## Итерация по аргументам

```bash
#!/bin/bash

# По всем аргументам скрипта
for arg in "$@"; do
    echo "Аргумент: $arg"
done

# По аргументам функции
process_files() {
    for file in "$@"; do
        echo "Обработка: $file"
    done
}

process_files "file1.txt" "file2.txt" "file 3.txt"
```

## Итерация по массиву

```bash
#!/bin/bash

fruits=("apple" "banana" "cherry" "date")

# По значениям
for fruit in "${fruits[@]}"; do
    echo "Фрукт: $fruit"
done

# По индексам
for i in "${!fruits[@]}"; do
    echo "fruits[$i] = ${fruits[$i]}"
done
```

## Итерация по строке

```bash
#!/bin/bash

# Разделение по пробелам
text="one two three"
for word in $text; do
    echo "Слово: $word"
done

# С изменением IFS
PATH_DIRS="$PATH"
IFS=':'
for dir in $PATH_DIRS; do
    echo "Директория: $dir"
done
IFS=$' \t\n'  # Восстановление
```

## Итерация по выводу команды

```bash
#!/bin/bash

# По строкам вывода (осторожно с пробелами!)
for line in $(cat file.txt); do
    echo "$line"
done

# Лучше использовать while для файлов
while IFS= read -r line; do
    echo "$line"
done < file.txt

# По результатам find
for file in $(find /tmp -name "*.log"); do
    echo "Лог: $file"
done

# Безопаснее с null-разделителем
while IFS= read -r -d '' file; do
    echo "Лог: $file"
done < <(find /tmp -name "*.log" -print0)
```

## Практические примеры

### Переименование файлов

```bash
#!/bin/bash

# Добавить префикс
for file in *.txt; do
    [[ -e "$file" ]] || continue
    mv "$file" "backup_$file"
done

# Изменить расширение
for file in *.jpeg; do
    [[ -e "$file" ]] || continue
    mv "$file" "${file%.jpeg}.jpg"
done
```

### Массовое создание директорий

```bash
#!/bin/bash

for project in project{1..5}; do
    mkdir -p "$project"/{src,tests,docs}
    touch "$project/README.md"
done
```

### Обработка серверов

```bash
#!/bin/bash

servers=("web01" "web02" "db01" "cache01")

for server in "${servers[@]}"; do
    echo "Проверка $server..."
    if ping -c 1 -W 1 "$server" &>/dev/null; then
        echo "  $server: OK"
    else
        echo "  $server: НЕДОСТУПЕН"
    fi
done
```

### Суммирование размеров файлов

```bash
#!/bin/bash

total=0
for file in *.log; do
    [[ -f "$file" ]] || continue
    size=$(stat -f %z "$file" 2>/dev/null || stat -c %s "$file")
    ((total += size))
    echo "$file: $size bytes"
done
echo "Всего: $total bytes"
```

### Генерация HTML

```bash
#!/bin/bash

echo "<ul>"
for item in "Главная" "О нас" "Контакты"; do
    echo "  <li>$item</li>"
done
echo "</ul>"
```

## Вложенные циклы

```bash
#!/bin/bash

# Таблица умножения
for i in {1..10}; do
    for j in {1..10}; do
        printf "%4d" $((i * j))
    done
    echo
done
```

## break и continue

```bash
#!/bin/bash

# break - выход из цикла
for i in {1..10}; do
    if [[ $i -eq 5 ]]; then
        echo "Прерывание на $i"
        break
    fi
    echo $i
done

# continue - пропуск итерации
for i in {1..10}; do
    if [[ $((i % 2)) -eq 0 ]]; then
        continue  # Пропускаем чётные
    fi
    echo $i
done
```

## Пустой список

```bash
#!/bin/bash

# Если список пуст, тело цикла не выполняется
empty_array=()
for item in "${empty_array[@]}"; do
    echo "Это не выполнится"
done
echo "Цикл пройден"
```
