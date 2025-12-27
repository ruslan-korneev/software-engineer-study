# pr и printf - форматированный вывод

## pr - подготовка файлов к печати

### Что такое pr?

**pr** (print) форматирует текстовые файлы для печати, добавляя заголовки, номера страниц и разбивая на страницы.

### Базовое использование

```bash
# Базовый вывод с заголовком
pr file.txt

# Вывод с кастомным заголовком
pr -h "My Document" file.txt

# Двойной интервал
pr -d file.txt
```

### Страницы и колонки

```bash
# Длина страницы (строк)
pr -l 50 file.txt       # 50 строк на страницу

# Ширина страницы
pr -w 80 file.txt       # 80 символов ширина

# Несколько колонок
pr -2 file.txt          # 2 колонки
pr -3 file.txt          # 3 колонки
pr -4 file.txt          # 4 колонки

# Колонки поперёк (across)
pr -a -3 file.txt       # заполнять по строкам, не по столбцам

# Объединить файлы в колонки
pr -m file1.txt file2.txt file3.txt
```

### Нумерация

```bash
# Нумерация строк
pr -n file.txt

# Формат нумерации
pr -n5 file.txt         # 5-значные номера

# Разделитель
pr -n: file.txt         # номер:строка
```

### Поля и отступы

```bash
# Отступ слева
pr -o 5 file.txt        # 5 пробелов слева

# Без заголовка и хвоста
pr -t file.txt

# Только определённые страницы
pr +5 file.txt          # начать с 5-й страницы
pr +5:10 file.txt       # страницы 5-10
```

### Разделитель колонок

```bash
# Разделитель между колонками
pr -s: -3 file.txt      # разделитель :
pr -s'|' -2 file.txt    # разделитель |

# Табуляция по умолчанию
pr -3 file.txt
```

### Практические примеры

```bash
# Подготовка кода к печати
pr -n -h "source.py" source.py | lpr

# Создание 2-колоночного документа
pr -2 -w 130 document.txt > two_column.txt

# Объединить два файла бок о бок
pr -m -t file1.txt file2.txt

# Печать с датой и нумерацией
pr -l 60 -h "Report $(date +%Y-%m-%d)" report.txt
```

## printf - форматированный вывод

### Что такое printf?

**printf** выводит форматированный текст по шаблону. Пришёл из языка C и широко используется в shell скриптах.

### Базовый синтаксис

```bash
printf "format_string" arguments
```

### Спецификаторы формата

```bash
# Строки
printf "%s\n" "Hello"           # Hello

# Целые числа
printf "%d\n" 42                # 42
printf "%i\n" 42                # 42 (то же)

# Числа с плавающей точкой
printf "%f\n" 3.14159           # 3.141590
printf "%.2f\n" 3.14159         # 3.14

# Восьмеричные
printf "%o\n" 255               # 377

# Шестнадцатеричные
printf "%x\n" 255               # ff
printf "%X\n" 255               # FF

# Символ
printf "%c\n" 65                # A
printf "%c\n" A                 # A

# Процент
printf "%%\n"                   # %
```

### Ширина и выравнивание

```bash
# Минимальная ширина
printf "%10s\n" "hello"         # "     hello"
printf "%-10s\n" "hello"        # "hello     "

# Для чисел
printf "%5d\n" 42               # "   42"
printf "%-5d\n" 42              # "42   "
printf "%05d\n" 42              # "00042"

# Знак
printf "%+d\n" 42               # +42
printf "%+d\n" -42              # -42
```

### Точность

```bash
# Для float - знаки после точки
printf "%.2f\n" 3.14159         # 3.14
printf "%.4f\n" 3.14159         # 3.1416

# Для строк - максимальная длина
printf "%.5s\n" "Hello World"   # Hello

# Комбинация ширины и точности
printf "%10.2f\n" 3.14159       # "      3.14"
printf "%-10.2f\n" 3.14159      # "3.14      "
```

### Escape-последовательности

```bash
\n  - новая строка
\t  - табуляция
\r  - возврат каретки
\\  - обратный слеш
\"  - кавычка
\NNN - восьмеричный код
\xNN - шестнадцатеричный код
```

```bash
printf "Line1\nLine2\n"
# Line1
# Line2

printf "Col1\tCol2\tCol3\n"
# Col1    Col2    Col3

printf "Path: C:\\Users\n"
# Path: C:\Users
```

### Практические примеры

#### Таблицы

```bash
# Заголовок таблицы
printf "%-20s %10s %10s\n" "Name" "Price" "Quantity"
printf "%-20s %10.2f %10d\n" "Apple" 1.99 50
printf "%-20s %10.2f %10d\n" "Banana" 0.99 100
printf "%-20s %10.2f %10d\n" "Orange" 2.49 30

# Name                      Price   Quantity
# Apple                      1.99         50
# Banana                     0.99        100
# Orange                     2.49         30
```

#### Прогресс-бар

```bash
# Простой прогресс
for i in {1..10}; do
    printf "\rProgress: %d%%" $((i*10))
    sleep 0.5
done
printf "\n"

# Графический прогресс
for i in {1..50}; do
    printf "\r[%-50s] %d%%" "$(printf '#%.0s' $(seq 1 $i))" $((i*2))
    sleep 0.1
done
printf "\n"
```

#### Цвета

```bash
# ANSI цвета
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

printf "${RED}Error${NC}: Something went wrong\n"
printf "${GREEN}Success${NC}: Operation completed\n"
```

#### Форматирование чисел

```bash
# Денежные суммы
printf "$%'.2f\n" 1234567.89
# $1,234,567.89 (локаль-зависимо)

# Номера телефонов
num=1234567890
printf "(%s) %s-%s\n" "${num:0:3}" "${num:3:3}" "${num:6:4}"
# (123) 456-7890

# Ведущие нули
for i in {1..10}; do
    printf "file_%03d.txt\n" $i
done
# file_001.txt ... file_010.txt
```

### printf в скриптах

```bash
#!/bin/bash

# Функция для логирования
log() {
    printf "[%s] %s\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$1"
}

log "Starting process"
log "Processing data"
log "Done"

# [2024-01-15 10:30:45] Starting process
# [2024-01-15 10:30:45] Processing data
# [2024-01-15 10:30:45] Done
```

```bash
# Генерация HTML
printf '<!DOCTYPE html>\n<html>\n<head>\n<title>%s</title>\n</head>\n<body>\n<h1>%s</h1>\n</body>\n</html>\n' \
    "My Page" "Welcome"
```

## echo vs printf

| Особенность | echo | printf |
|-------------|------|--------|
| Переносимость | Разное поведение | Стандартизирован |
| Форматирование | Ограниченное | Полное |
| Escape-последовательности | С -e | Всегда |
| Без переноса строки | echo -n | Не добавлять \n |

```bash
# echo -e для escape
echo -e "Line1\nLine2"

# printf всегда понимает escape
printf "Line1\nLine2\n"

# Лучше использовать printf в скриптах
```

## Резюме

### pr

| Опция | Описание |
|-------|----------|
| `-h "text"` | Заголовок |
| `-l N` | Строк на странице |
| `-w N` | Ширина |
| `-2, -3...` | Колонки |
| `-m` | Объединить файлы |
| `-n` | Нумерация |
| `-t` | Без заголовка |
| `-o N` | Отступ слева |

### printf

| Спецификатор | Описание |
|--------------|----------|
| `%s` | Строка |
| `%d`, `%i` | Целое число |
| `%f` | Float |
| `%x`, `%X` | Hex |
| `%o` | Octal |
| `%c` | Символ |
| `%%` | Процент |
| `%Ns` | Ширина N |
| `%-Ns` | Выравнивание влево |
| `%0Nd` | Ведущие нули |
| `%.Nf` | N знаков после точки |
