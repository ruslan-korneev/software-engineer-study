# Группировка команд

## Зачем группировать команды

Группировка позволяет:
- Перенаправлять ввод/вывод для нескольких команд сразу
- Выполнять несколько команд в одном контексте
- Управлять подпроцессами

## Два способа группировки

### Фигурные скобки { }

Выполняются в **текущем shell**:

```bash
{ command1; command2; command3; }
```

**Важно:**
- Пробел после `{` обязателен
- Точка с запятой перед `}` обязательна
- Или каждая команда на новой строке

```bash
# Корректно
{ echo "one"; echo "two"; }

# Корректно
{
    echo "one"
    echo "two"
}

# ОШИБКА - нет пробела после {
{echo "one"; echo "two"; }

# ОШИБКА - нет ; перед }
{ echo "one"; echo "two" }
```

### Круглые скобки ( )

Выполняются в **подоболочке (subshell)**:

```bash
( command1; command2; command3 )
```

```bash
# Корректно
( echo "one"; echo "two" )

# Последняя точка с запятой не обязательна
```

## Разница между { } и ( )

### Влияние на переменные

```bash
#!/bin/bash

var="original"

# Фигурные скобки - текущий shell
{ var="changed"; echo "Inside: $var"; }
echo "Outside: $var"  # changed

var="original"

# Круглые скобки - subshell
( var="changed"; echo "Inside: $var" )
echo "Outside: $var"  # original (не изменилось!)
```

### Влияние на рабочую директорию

```bash
#!/bin/bash

pwd  # /home/user

# Фигурные скобки
{ cd /tmp; pwd; }  # /tmp
pwd  # /tmp (изменилась!)

# Круглые скобки
( cd /etc; pwd )   # /etc
pwd  # /tmp (осталась прежней)
```

## Перенаправление для группы

### Общий вывод

```bash
#!/bin/bash

# Весь вывод в файл
{
    echo "=== Системная информация ==="
    date
    uname -a
    df -h
} > system_info.txt

# Добавление
{
    echo ""
    echo "=== Память ==="
    free -h
} >> system_info.txt
```

### Общий ввод

```bash
#!/bin/bash

# Чтение нескольких значений из файла
{
    read line1
    read line2
    read line3
} < input.txt

echo "Line 1: $line1"
echo "Line 2: $line2"
echo "Line 3: $line3"
```

### Комбинирование

```bash
#!/bin/bash

# Преобразование данных
{
    while read line; do
        echo "${line^^}"  # В верхний регистр
    done
} < input.txt > output.txt
```

## Практические примеры

### Транзакционная операция

```bash
#!/bin/bash

# Все команды должны выполниться или ни одна
(
    set -e  # Выход при ошибке (только в subshell)
    cp important.txt backup/
    rm important.txt.old
    mv new_data.txt important.txt
) || {
    echo "Ошибка! Откат изменений..."
    # Восстановление
}
```

### Изоляция изменений окружения

```bash
#!/bin/bash

# Временное изменение PATH только для группы команд
(
    export PATH="/custom/bin:$PATH"
    export DEBUG=true
    ./my_script.sh
)
# PATH и DEBUG не изменились
```

### Логирование с временной меткой

```bash
#!/bin/bash

log_with_timestamp() {
    {
        while IFS= read -r line; do
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] $line"
        done
    }
}

{
    echo "Начало скрипта"
    ls -la
    echo "Конец скрипта"
} 2>&1 | log_with_timestamp > script.log
```

### Генерация отчёта

```bash
#!/bin/bash

generate_report() {
    {
        echo "=== ОТЧЁТ ==="
        echo "Дата: $(date)"
        echo ""
        echo "=== Диск ==="
        df -h
        echo ""
        echo "=== Память ==="
        free -h
        echo ""
        echo "=== Процессы ==="
        ps aux --sort=-%mem | head -10
    }
}

generate_report > report.txt
```

### Pipe с группой команд

```bash
#!/bin/bash

# Обработка данных несколькими командами
cat data.txt | {
    count=0
    sum=0
    while read value; do
        ((count++))
        ((sum += value))
    done
    echo "Count: $count"
    echo "Sum: $sum"
    echo "Average: $((sum / count))"
}
```

### Фоновое выполнение группы

```bash
#!/bin/bash

# Группа команд в фоне
{
    sleep 5
    echo "Задача 1 завершена"
    sleep 5
    echo "Задача 2 завершена"
} &

echo "Фоновая группа запущена"
wait
echo "Всё завершено"
```

## Вложенные группы

```bash
#!/bin/bash

{
    echo "Внешний уровень"
    {
        echo "Внутренний уровень"
    }
}

# Subshell внутри группы
{
    echo "Начало"
    (
        cd /tmp
        ls
    )
    pwd  # Остались в исходной директории
    echo "Конец"
}
```

## Использование с условиями

```bash
#!/bin/bash

# Условное выполнение группы
condition=true

$condition && {
    echo "Условие истинно"
    echo "Выполняем действия"
}

# С || для обработки ошибок
{
    command1
    command2
} || {
    echo "Ошибка!"
    exit 1
}
```

## Когда использовать { } vs ( )

### Используйте { } когда:
- Нужно сохранить изменения переменных
- Нужно сохранить изменение директории
- Важна производительность (нет создания процесса)

### Используйте ( ) когда:
- Нужна изоляция изменений
- Временные изменения окружения
- Безопасное выполнение с `set -e`
- Параллельное выполнение
