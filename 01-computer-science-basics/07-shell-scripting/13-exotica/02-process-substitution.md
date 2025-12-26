# Подстановка процессов (Process Substitution)

## Что такое Process Substitution

Process Substitution позволяет использовать вывод команды как файл. Это решает проблемы, когда программа ожидает файл, а не stdin.

## Синтаксис

```bash
# Вывод команды как файл для чтения
<(command)

# Файл для записи (результат попадает на stdin команды)
>(command)
```

## <(command) - вывод как файл

```bash
#!/bin/bash

# diff ожидает два файла
# Сравнение вывода двух команд
diff <(ls dir1) <(ls dir2)

# Сравнение отсортированного вывода
diff <(sort file1.txt) <(sort file2.txt)
```

### Как это работает

```bash
# <(command) создаёт временный файловый дескриптор
echo <(echo "hello")
# /dev/fd/63 (или подобный путь)

# Можно читать как обычный файл
cat <(echo "hello world")
# hello world
```

## Практические примеры

### Сравнение файлов

```bash
#!/bin/bash

# Сравнение содержимого двух директорий
diff <(ls -la /dir1) <(ls -la /dir2)

# Сравнение без учёта порядка строк
diff <(sort file1.txt) <(sort file2.txt)

# Сравнение вывода команд
diff <(ps aux | grep nginx) <(cat saved_processes.txt)
```

### Чтение из нескольких источников

```bash
#!/bin/bash

# paste объединяет строки из файлов
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f7 /etc/passwd)
# Выводит: username    shell
```

### Передача в цикл while

```bash
#!/bin/bash

# Проблема с pipe: переменные в subshell
count=0
cat file.txt | while read line; do
    ((count++))
done
echo $count  # 0! (переменная в subshell)

# Решение с process substitution
count=0
while read line; do
    ((count++))
done < <(cat file.txt)
echo $count  # Корректное значение
```

### Объединение данных

```bash
#!/bin/bash

# Объединение двух отсортированных списков
sort -m <(sort file1.txt) <(sort file2.txt)

# Join требует отсортированные файлы
join <(sort file1.txt) <(sort file2.txt)
```

## >(command) - файл для записи

```bash
#!/bin/bash

# Вывод в несколько мест одновременно
echo "Hello" | tee >(cat > file1.txt) >(cat > file2.txt)

# Логирование с обработкой
command 2> >(while read line; do
    echo "[ERROR] $line" >> error.log
done)
```

### Параллельная обработка

```bash
#!/bin/bash

# Отправка данных нескольким обработчикам
generate_data | tee \
    >(process1 > output1.txt) \
    >(process2 > output2.txt) \
    >(process3 > output3.txt) \
    > /dev/null
```

## Комбинирование с другими техниками

### С группировкой команд

```bash
#!/bin/bash

diff <({
    echo "=== File 1 ==="
    cat file1.txt
}) <({
    echo "=== File 2 ==="
    cat file2.txt
})
```

### С массивами

```bash
#!/bin/bash

# Чтение в массив из команды
mapfile -t lines < <(find /tmp -name "*.txt")

for file in "${lines[@]}"; do
    echo "Found: $file"
done
```

### Сложная обработка

```bash
#!/bin/bash

# Обработка логов с фильтрацией
tail -f /var/log/syslog | tee \
    >(grep --line-buffered ERROR >> errors.log) \
    >(grep --line-buffered WARN >> warnings.log) \
    > /dev/null
```

## Преимущества перед временными файлами

```bash
#!/bin/bash

# Без process substitution (временные файлы)
sort file1.txt > /tmp/sorted1.txt
sort file2.txt > /tmp/sorted2.txt
diff /tmp/sorted1.txt /tmp/sorted2.txt
rm /tmp/sorted1.txt /tmp/sorted2.txt

# С process substitution (чище и безопаснее)
diff <(sort file1.txt) <(sort file2.txt)
```

## Примеры использования

### Проверка изменений конфигурации

```bash
#!/bin/bash

# Сравнение текущей конфигурации с бэкапом
diff <(cat /etc/nginx/nginx.conf) <(cat /backup/nginx.conf.bak)
```

### Анализ логов

```bash
#!/bin/bash

# Уникальные IP за сегодня vs вчера
diff \
    <(grep "$(date +%Y-%m-%d)" access.log | awk '{print $1}' | sort -u) \
    <(grep "$(date -d yesterday +%Y-%m-%d)" access.log | awk '{print $1}' | sort -u)
```

### Мониторинг процессов

```bash
#!/bin/bash

# Сохранение снимка процессов
baseline=$(cat < <(ps aux))

sleep 60

# Сравнение с текущим состоянием
diff <(echo "$baseline") <(ps aux)
```

### Обработка данных из URL

```bash
#!/bin/bash

# Сравнение данных из двух API
diff \
    <(curl -s https://api.example.com/v1/data | jq '.items') \
    <(curl -s https://api.example.com/v2/data | jq '.items')
```

## Ограничения

```bash
# Process substitution не работает в sh (только bash, zsh, ksh)
#!/bin/sh
cat <(echo "hello")  # Ошибка!

# Нельзя использовать для записи в некоторых командах
# Файл может быть прочитан только один раз

# Не все программы работают с /dev/fd/*
```

## Проверка поддержки

```bash
#!/bin/bash

# Проверка, что мы в bash
if [[ -n "$BASH_VERSION" ]]; then
    # Process substitution доступен
    cat <(echo "Works!")
fi
```
