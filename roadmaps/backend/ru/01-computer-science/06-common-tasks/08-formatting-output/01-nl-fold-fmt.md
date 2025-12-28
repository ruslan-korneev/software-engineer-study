# nl, fold, fmt - форматирование текста

[prev: 05-sed](../07-text-processing/05-sed.md) | [next: 02-pr-printf](./02-pr-printf.md)

---
## nl - нумерация строк

### Что такое nl?

**nl** (number lines) - утилита для нумерации строк в файле. Более гибкая чем `cat -n`.

### Базовое использование

```bash
# Нумеровать непустые строки
nl file.txt
#      1  First line
#      2  Second line
#
#      3  Third line

# Нумеровать все строки
nl -ba file.txt
#      1  First line
#      2  Second line
#      3
#      4  Third line
```

### Стили нумерации (-b)

```bash
# -b стиль для тела документа
nl -ba file.txt    # all - все строки
nl -bt file.txt    # text - только непустые (по умолчанию)
nl -bn file.txt    # none - не нумеровать
nl -bp'pattern' file.txt  # только строки соответствующие pattern

# Примеры
nl -bp'^#' file.txt    # только строки начинающиеся с #
nl -bp'TODO' file.txt  # только строки с TODO
```

### Формат нумерации (-n)

```bash
# -n формат номера
nl -nln file.txt    # выровнять по левому краю
nl -nrn file.txt    # выровнять по правому краю (по умолчанию)
nl -nrz file.txt    # с ведущими нулями

# Примеры:
# -nln:  1  text
# -nrn:     1  text
# -nrz: 000001  text
```

### Ширина и разделитель

```bash
# Ширина поля номера (-w)
nl -w3 file.txt     # ширина 3 символа
#   1  text
#   2  text

# Разделитель после номера (-s)
nl -s': ' file.txt
#      1: text
#      2: text

nl -s'.' -w4 file.txt
#    1.text
```

### Начальный номер и шаг

```bash
# Начать с номера (-v)
nl -v 100 file.txt
#    100  text
#    101  text

# Шаг нумерации (-i)
nl -i 10 file.txt
#      1  text
#     11  text
#     21  text
```

### Секции документа

nl может работать с секциями (header, body, footer):

```bash
# Маркеры секций:
# \:\:\:  - header
# \:\:    - body (по умолчанию)
# \:      - footer

# Настройки для секций
nl -ha -ba -fa file.txt   # нумеровать все секции
```

## fold - переносы строк

### Что такое fold?

**fold** разбивает длинные строки на части указанной ширины.

### Базовое использование

```bash
# Разбить на строки по 80 символов (по умолчанию)
fold file.txt

# Указать ширину
fold -w 40 file.txt

# Разбить по словам (не разрывать слова)
fold -s file.txt
fold -w 40 -s file.txt
```

### Примеры

```bash
# Длинная строка
echo "This is a very long line that needs to be wrapped to fit on the screen" | fold -w 30
# This is a very long line tha
# t needs to be wrapped to fit
#  on the screen

# С учётом слов
echo "This is a very long line that needs to be wrapped" | fold -w 30 -s
# This is a very long line
# that needs to be wrapped

# По байтам (для многобайтовых кодировок)
fold -b -w 80 file.txt
```

### Практические применения

```bash
# Форматирование длинных строк для email
fold -s -w 72 letter.txt > formatted_letter.txt

# Подготовка текста для терминала
cat README.md | fold -s -w $(tput cols)

# Разбиение base64
cat file | base64 | fold -w 76
```

## fmt - форматирование абзацев

### Что такое fmt?

**fmt** переформатирует текст, объединяя короткие строки и разбивая длинные.

### Базовое использование

```bash
# Форматировать с шириной 75 (по умолчанию)
fmt file.txt

# Указать ширину
fmt -w 60 file.txt

# Указать целевую ширину (пытается достичь)
fmt -g 50 file.txt   # goal
```

### Опции fmt

```bash
# Сохранить структуру
fmt -u file.txt      # uniform spacing (один пробел между словами)
fmt -c file.txt      # crown margin (сохранить отступ первой строки)
fmt -t file.txt      # tagged paragraphs (отступ первой строки)
fmt -s file.txt      # split only (только разбивать, не объединять)

# Ширина
fmt -w 80 file.txt              # максимальная ширина
fmt -g 65 -w 80 file.txt        # целевая 65, максимум 80
```

### Сравнение fold и fmt

```bash
# fold - просто режет по символам
# fmt - умно переформатирует

# Исходный текст:
# Short line.
# Another very long line that extends beyond the normal width.

# fold -w 40 -s:
# Short line.
# Another very long line that
# extends beyond the normal width.

# fmt -w 40:
# Short line.  Another very long line
# that extends beyond the normal width.
```

### Практические примеры

```bash
# Переформатировать readme
fmt -w 80 README.txt > README_formatted.txt

# Форматирование email
fmt -w 72 -s letter.txt

# Переформатировать абзац (из clipboard)
xclip -o | fmt -w 80 | xclip

# Исправить форматирование документа
fmt -u -w 78 document.txt > fixed.txt
```

## Комбинирование инструментов

### Нумерация и форматирование

```bash
# Пронумеровать и обрезать длинные строки
nl file.txt | fold -w 80

# Сначала форматировать, потом нумеровать
fmt -w 70 file.txt | nl

# Подготовка документа для печати
fmt -w 72 document.txt | nl -ba | pr -h "My Document"
```

### Обработка кода

```bash
# Нумерация только строк с кодом
nl -ba source.py

# Нумерация с шириной под большие файлы
nl -w 6 -nrz large_file.py
# 000001  code
# 000002  more code
```

### Создание отчётов

```bash
# Добавить номера строк к логам
nl -w 8 -s ' | ' app.log

# Форматировать вывод
some_command | fmt -w 100 | nl
```

## Альтернативы

### pr - подготовка к печати

```bash
# Добавить заголовок и номера страниц
pr file.txt

# С пользовательским заголовком
pr -h "My Document" file.txt

# Несколько колонок
pr -2 file.txt    # 2 колонки
pr -3 file.txt    # 3 колонки
```

### column - колоночный вывод

```bash
# Выровнять в колонки
column file.txt

# С разделителем
column -t -s',' data.csv

# JSON pretty print (некоторые версии)
echo '{"a":1}' | column -t -s':'
```

### expand/unexpand - табы и пробелы

```bash
# Табы в пробелы
expand file.txt

# Пробелы в табы
unexpand file.txt

# Указать шаг табуляции
expand -t 4 file.txt
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `nl file` | Нумеровать непустые строки |
| `nl -ba file` | Нумеровать все строки |
| `nl -w4 -s': '` | Ширина 4, разделитель ": " |
| `nl -v 100` | Начать с 100 |
| `fold -w 80` | Разбить на 80 символов |
| `fold -s` | Разбивать по словам |
| `fmt -w 75` | Переформатировать с шириной 75 |
| `fmt -s` | Только разбивать строки |
| `fmt -u` | Нормализовать пробелы |

---

[prev: 05-sed](../07-text-processing/05-sed.md) | [next: 02-pr-printf](./02-pr-printf.md)
