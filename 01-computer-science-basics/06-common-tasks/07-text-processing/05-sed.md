# sed - потоковый редактор

## Что такое sed?

**sed** (stream editor) - мощный инструмент для обработки текстовых потоков. Позволяет выполнять сложные преобразования текста без интерактивного редактирования.

## Базовый синтаксис

```bash
sed [опции] 'команда' файл
sed [опции] -e 'команда1' -e 'команда2' файл
sed [опции] -f script.sed файл
```

## Команда замены (s)

### Базовая замена

```bash
# Синтаксис: s/pattern/replacement/flags

# Заменить первое вхождение в каждой строке
sed 's/old/new/' file.txt

# Заменить все вхождения в строке (g = global)
sed 's/old/new/g' file.txt

# Заменить N-е вхождение
sed 's/old/new/2' file.txt    # второе
sed 's/old/new/3g' file.txt   # с третьего и далее
```

### Регистронезависимая замена

```bash
# Флаг i (или I)
sed 's/error/ERROR/gi' file.txt
```

### Разделители

```bash
# Можно использовать другие разделители (удобно для путей)
sed 's|/usr/local|/opt|g' file.txt
sed 's#http://#https://#g' file.txt
sed 's@old@new@g' file.txt
```

### Обратные ссылки

```bash
# & - вся совпавшая строка
sed 's/[0-9]\+/[&]/g' file.txt
# 123 -> [123]

# \1, \2... - группы захвата
sed 's/\(hello\) \(world\)/\2 \1/g' file.txt
# hello world -> world hello

# Поменять формат даты
sed 's|\([0-9]\{2\}\)/\([0-9]\{2\}\)/\([0-9]\{4\}\)|\3-\1-\2|g' dates.txt
# 01/15/2024 -> 2024-01-15
```

## Адресация (выбор строк)

### По номеру строки

```bash
# Конкретная строка
sed '5s/old/new/' file.txt      # только 5-я строка

# Диапазон строк
sed '5,10s/old/new/' file.txt   # строки 5-10

# С N до конца
sed '5,$s/old/new/' file.txt

# Первая строка
sed '1s/old/new/' file.txt

# Последняя строка
sed '$s/old/new/' file.txt
```

### По шаблону

```bash
# Строки содержащие pattern
sed '/error/s/old/new/' file.txt

# Строки НЕ содержащие pattern
sed '/error/!s/old/new/' file.txt

# От одного pattern до другого
sed '/START/,/END/s/old/new/' file.txt
```

### Комбинации

```bash
# Номер + шаблон
sed '1,/END/s/old/new/' file.txt    # от 1 строки до END

# Каждая N-я строка
sed '0~2s/old/new/' file.txt        # каждая чётная (2,4,6...)
sed '1~2s/old/new/' file.txt        # каждая нечётная (1,3,5...)
```

## Другие команды

### Удаление (d)

```bash
# Удалить строку
sed '5d' file.txt               # 5-ю строку
sed '5,10d' file.txt            # строки 5-10
sed '/pattern/d' file.txt       # строки с pattern
sed '/^$/d' file.txt            # пустые строки
sed '/^#/d' file.txt            # комментарии

# Удалить всё кроме
sed '/pattern/!d' file.txt      # оставить только с pattern
```

### Печать (p)

```bash
# По умолчанию sed печатает все строки
# -n отключает автопечать

# Показать только совпадающие строки (как grep)
sed -n '/pattern/p' file.txt

# Показать диапазон
sed -n '5,10p' file.txt

# Показать с номерами строк
sed -n '=' file.txt | paste - file.txt
```

### Вставка (i, a)

```bash
# Вставить ПЕРЕД строкой (i = insert)
sed '3i\New line before line 3' file.txt

# Вставить ПОСЛЕ строки (a = append)
sed '3a\New line after line 3' file.txt

# После каждой строки с pattern
sed '/pattern/a\Added after pattern' file.txt

# В начало файла
sed '1i\Header line' file.txt

# В конец файла
sed '$a\Footer line' file.txt
```

### Замена строки (c)

```bash
# Заменить всю строку
sed '5c\Completely new line 5' file.txt

# Заменить строки с pattern
sed '/old line/c\new line' file.txt
```

### Преобразование регистра

```bash
# В верхний регистр
sed 's/.*/\U&/' file.txt
sed 's/[a-z]/\U&/g' file.txt

# В нижний регистр
sed 's/.*/\L&/' file.txt

# Первую букву заглавной
sed 's/\b\(.\)/\U\1/g' file.txt

# Комбинация
sed 's/\(.\{1\}\)\(.*\)/\U\1\L\2/' file.txt  # Hello world -> Hello world
```

## Опции sed

```bash
# Редактирование на месте (-i)
sed -i 's/old/new/g' file.txt

# С бекапом
sed -i.bak 's/old/new/g' file.txt

# Несколько команд (-e)
sed -e 's/a/b/' -e 's/c/d/' file.txt

# Из файла скрипта (-f)
sed -f commands.sed file.txt

# Extended regex (-E или -r)
sed -E 's/[0-9]+/NUMBER/g' file.txt

# Без автопечати (-n)
sed -n '/pattern/p' file.txt

# Показать команды выполнения (debug)
sed --debug 's/old/new/' file.txt
```

## Практические примеры

### Редактирование конфигов

```bash
# Изменить значение параметра
sed -i 's/^port=.*/port=8080/' config.ini

# Раскомментировать строку
sed -i 's/^#\(ServerName\)/\1/' apache.conf

# Закомментировать строку
sed -i 's/^ServerName/#&/' apache.conf

# Добавить строку после совпадения
sed -i '/\[section\]/a\new_option=value' config.ini
```

### Очистка данных

```bash
# Удалить пробелы в начале/конце строки
sed 's/^[[:space:]]*//' file.txt    # в начале
sed 's/[[:space:]]*$//' file.txt    # в конце
sed 's/^[[:space:]]*//;s/[[:space:]]*$//' file.txt  # оба

# Удалить пустые строки
sed '/^$/d' file.txt

# Удалить строки только с пробелами
sed '/^[[:space:]]*$/d' file.txt

# Сжать множественные пробелы
sed 's/  */ /g' file.txt
```

### Работа с HTML/XML

```bash
# Удалить HTML теги
sed 's/<[^>]*>//g' file.html

# Извлечь содержимое тега
sed -n 's/.*<title>\(.*\)<\/title>.*/\1/p' file.html

# Заменить сущности
sed 's/&amp;/\&/g; s/&lt;/</g; s/&gt;/>/g' file.html
```

### Обработка логов

```bash
# Извлечь IP адреса
sed -n 's/.*\([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\).*/\1/p' access.log

# Убрать временные метки
sed 's/^\[[^]]*\] //' log.txt

# Заменить пути
sed 's|/var/www/old|/var/www/new|g' config.txt
```

### Работа с CSV

```bash
# Заменить разделитель
sed 's/,/;/g' file.csv

# Удалить первую колонку
sed 's/^[^,]*,//' file.csv

# Добавить колонку
sed 's/$/,new_value/' file.csv

# Удалить кавычки
sed 's/"//g' file.csv
```

### Многострочная обработка

```bash
# Объединить строки (N - добавить следующую)
sed 'N;s/\n/ /' file.txt

# Удалить переносы в многострочных значениях
sed ':a;N;$!ba;s/\\\n//g' file.txt

# Заменить многострочный блок
sed '/START/,/END/{s/old/new/g}' file.txt
```

## Скрипты sed

Сохранить команды в файл:

```bash
# commands.sed
s/foo/bar/g
/^#/d
s/^[[:space:]]*//
```

```bash
# Использование
sed -f commands.sed input.txt
```

## Сравнение с другими инструментами

| Задача | sed | Альтернатива |
|--------|-----|--------------|
| Простая замена | `sed 's/a/b/g'` | `tr 'a' 'b'` |
| Сложная замена | `sed 's/pat/rep/g'` | `perl -pe` |
| Удаление строк | `sed '/pat/d'` | `grep -v` |
| Поиск строк | `sed -n '/pat/p'` | `grep` |
| Колонки | сложно | `awk`, `cut` |

## Резюме команд

| Команда | Описание |
|---------|----------|
| `s/old/new/` | Заменить первое |
| `s/old/new/g` | Заменить все |
| `s/old/new/i` | Без регистра |
| `/pat/s/old/new/` | Только в строках с pat |
| `5s/old/new/` | Только в 5-й строке |
| `5,10s/old/new/` | В строках 5-10 |
| `d` | Удалить строку |
| `p` | Напечатать строку |
| `i\text` | Вставить перед |
| `a\text` | Вставить после |
| `c\text` | Заменить строку |
| `-i` | Редактировать файл |
| `-n` | Без автопечати |
| `-E` | Extended regex |
