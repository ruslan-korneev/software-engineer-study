# Паттерны в case

## Типы паттернов

В операторе `case` можно использовать различные паттерны для сопоставления:

| Паттерн | Описание |
|---------|----------|
| `*` | Любая строка (включая пустую) |
| `?` | Любой один символ |
| `[abc]` | Любой символ из набора |
| `[a-z]` | Любой символ из диапазона |
| `[!abc]` или `[^abc]` | Любой символ НЕ из набора |
| `pattern1\|pattern2` | Альтернатива (ИЛИ) |

## Паттерн * (wildcard)

Соответствует любой строке:

```bash
case $filename in
    *.txt)
        echo "Текстовый файл"
        ;;
    *.sh)
        echo "Shell скрипт"
        ;;
    test*)
        echo "Файл начинается с 'test'"
        ;;
    *backup*)
        echo "Содержит 'backup' в имени"
        ;;
    *)
        echo "Что-то другое"
        ;;
esac
```

## Паттерн ? (один символ)

```bash
case $code in
    ???)
        echo "Трёхбуквенный код"
        ;;
    ??)
        echo "Двухбуквенный код"
        ;;
    ?)
        echo "Однобуквенный код"
        ;;
esac
```

## Символьные классы [...]

### Набор символов

```bash
case $answer in
    [yY])
        echo "Да (одна буква)"
        ;;
    [nN])
        echo "Нет (одна буква)"
        ;;
    [0-9])
        echo "Одна цифра"
        ;;
    [a-zA-Z])
        echo "Одна буква"
        ;;
esac
```

### Диапазоны

```bash
case $char in
    [a-z])
        echo "Строчная буква"
        ;;
    [A-Z])
        echo "Прописная буква"
        ;;
    [0-9])
        echo "Цифра"
        ;;
    [[:space:]])
        echo "Пробельный символ"
        ;;
    [[:punct:]])
        echo "Знак препинания"
        ;;
esac
```

### POSIX классы символов

```bash
case $char in
    [[:alpha:]])
        echo "Буква"
        ;;
    [[:digit:]])
        echo "Цифра"
        ;;
    [[:alnum:]])
        echo "Буква или цифра"
        ;;
    [[:lower:]])
        echo "Строчная буква"
        ;;
    [[:upper:]])
        echo "Прописная буква"
        ;;
    [[:space:]])
        echo "Пробел"
        ;;
esac
```

### Отрицание

```bash
case $char in
    [!0-9])
        echo "НЕ цифра"
        ;;
    [^a-z])
        echo "НЕ строчная буква"
        ;;
esac
```

## Альтернативы (|)

```bash
case $input in
    yes|y|Y|YES)
        echo "Положительный ответ"
        ;;
    no|n|N|NO)
        echo "Отрицательный ответ"
        ;;
    *.txt|*.text|*.TXT)
        echo "Текстовый файл"
        ;;
    [0-9]|[0-9][0-9])
        echo "Одна или две цифры"
        ;;
esac
```

## Комбинирование паттернов

```bash
case $filename in
    # Начинается с буквы и заканчивается цифрой
    [a-zA-Z]*[0-9])
        echo "Начинается с буквы, заканчивается цифрой"
        ;;

    # Ровно 3 символа: буква-цифра-буква
    [a-z][0-9][a-z])
        echo "Паттерн: буква-цифра-буква"
        ;;

    # Расширение из 3 букв
    *.???)
        echo "Расширение из 3 символов"
        ;;

    # README или readme с любым расширением
    [Rr][Ee][Aa][Dd][Mm][Ee]*)
        echo "Файл README"
        ;;
esac
```

## Практические примеры

### Валидация ввода

```bash
#!/bin/bash

validate_input() {
    case $1 in
        # Пустая строка
        "")
            echo "Ввод не может быть пустым"
            return 1
            ;;
        # Только цифры
        [0-9]|[0-9][0-9]|[0-9][0-9][0-9])
            echo "Корректное число (1-3 цифры)"
            return 0
            ;;
        # Почтовый индекс (6 цифр)
        [0-9][0-9][0-9][0-9][0-9][0-9])
            echo "Корректный почтовый индекс"
            return 0
            ;;
        # Начинается с цифры - ошибка для имён
        [0-9]*)
            echo "Имя не может начинаться с цифры"
            return 1
            ;;
        # Всё остальное
        *)
            echo "Некорректный ввод"
            return 1
            ;;
    esac
}
```

### Определение MIME-типа

```bash
#!/bin/bash

get_mime_type() {
    case $1 in
        # Текстовые файлы
        *.txt|*.text)
            echo "text/plain"
            ;;
        *.html|*.htm)
            echo "text/html"
            ;;
        *.css)
            echo "text/css"
            ;;
        *.js)
            echo "application/javascript"
            ;;
        *.json)
            echo "application/json"
            ;;
        *.xml)
            echo "application/xml"
            ;;

        # Изображения
        *.jpg|*.jpeg)
            echo "image/jpeg"
            ;;
        *.png)
            echo "image/png"
            ;;
        *.gif)
            echo "image/gif"
            ;;
        *.svg)
            echo "image/svg+xml"
            ;;

        # Архивы
        *.zip)
            echo "application/zip"
            ;;
        *.tar.gz|*.tgz)
            echo "application/gzip"
            ;;

        # По умолчанию
        *)
            echo "application/octet-stream"
            ;;
    esac
}
```

### Обработка расширений файлов

```bash
#!/bin/bash

process_file() {
    local file=$1

    case $file in
        # Исходный код
        *.[ch])
            echo "Файл C ($file)"
            gcc -c "$file"
            ;;
        *.cpp|*.cc|*.cxx)
            echo "Файл C++ ($file)"
            g++ -c "$file"
            ;;
        *.py)
            echo "Файл Python ($file)"
            python3 "$file"
            ;;
        *.sh)
            echo "Shell скрипт ($file)"
            bash "$file"
            ;;

        # Документы
        *.md|*.markdown)
            echo "Markdown ($file)"
            pandoc "$file" -o "${file%.md}.html"
            ;;

        # Игнорируемые файлы
        *.bak|*.tmp|*~)
            echo "Пропуск временного файла: $file"
            ;;

        # Скрытые файлы
        .*)
            echo "Скрытый файл: $file"
            ;;

        *)
            echo "Неизвестный тип: $file"
            ;;
    esac
}
```

### Парсинг URL

```bash
#!/bin/bash

parse_url() {
    local url=$1

    case $url in
        http://*)
            echo "Протокол: HTTP"
            echo "Хост: ${url#http://}"
            ;;
        https://*)
            echo "Протокол: HTTPS"
            echo "Хост: ${url#https://}"
            ;;
        ftp://*)
            echo "Протокол: FTP"
            ;;
        mailto:*)
            echo "Email: ${url#mailto:}"
            ;;
        /*)
            echo "Абсолютный путь"
            ;;
        ./*)
            echo "Относительный путь (текущая директория)"
            ;;
        ../*)
            echo "Относительный путь (родительская директория)"
            ;;
        *)
            echo "Относительный путь или домен"
            ;;
    esac
}
```

## Регистронезависимое сравнение

```bash
#!/bin/bash

# Вариант 1: Явное перечисление
case $answer in
    [yY]|[yY][eE][sS])
        echo "Да"
        ;;
esac

# Вариант 2: shopt nocasematch
shopt -s nocasematch
case $answer in
    yes)
        echo "Да"
        ;;
    no)
        echo "Нет"
        ;;
esac
shopt -u nocasematch

# Вариант 3: Преобразование к нижнему регистру
case ${answer,,} in
    yes)
        echo "Да"
        ;;
    no)
        echo "Нет"
        ;;
esac
```

## Extended Glob Patterns

С `shopt -s extglob` доступны расширенные паттерны:

```bash
#!/bin/bash
shopt -s extglob

case $filename in
    # Ноль или более вхождений
    *(ab))
        echo "Ноль или более 'ab'"
        ;;
    # Одно или более вхождений
    +(ab))
        echo "Одно или более 'ab'"
        ;;
    # Ноль или одно вхождение
    ?(ab))
        echo "Ноль или одно 'ab'"
        ;;
    # Ровно одно из перечисленных
    @(jpg|png|gif))
        echo "Одно из: jpg, png, gif"
        ;;
    # НЕ соответствует паттерну
    !(*.txt))
        echo "Не текстовый файл"
        ;;
esac
```
