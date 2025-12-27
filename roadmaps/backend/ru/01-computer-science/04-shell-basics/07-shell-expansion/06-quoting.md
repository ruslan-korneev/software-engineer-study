# Quoting (кавычки и экранирование)

## Зачем нужны кавычки?

Кавычки контролируют, какие раскрытия (expansions) shell выполняет:

- **Без кавычек** — все раскрытия активны
- **Двойные кавычки `""`** — большинство раскрытий работает
- **Одинарные кавычки `''`** — никаких раскрытий
- **Обратный слэш `\`** — экранирование одного символа

## Сравнение типов кавычек

| Символ | Без кавычек | Двойные `""` | Одинарные `''` |
|--------|-------------|--------------|----------------|
| `$VAR` | Раскрывается | Раскрывается | Литерально |
| `$(cmd)` | Раскрывается | Раскрывается | Литерально |
| `*`, `?` | Glob | Литерально | Литерально |
| `~` | Раскрывается | Литерально | Литерально |
| `{a,b}` | Раскрывается | Литерально | Литерально |
| Пробелы | Word splitting | Сохраняются | Сохраняются |

## Одинарные кавычки ('')

Всё внутри одинарных кавычек — буквальный текст:

```bash
$ echo '$HOME'
$HOME                    # не раскрылось

$ echo '$(date)'
$(date)                  # не выполнилось

$ echo 'Hello   World'
Hello   World            # пробелы сохранены

$ echo '*'
*                        # не glob
```

### Когда использовать
```bash
# Регулярные выражения
$ grep 'pattern with $pecial chars' file

# Фиксированные строки
$ echo 'No variables here'

# JSON/YAML
$ echo '{"key": "value"}'
```

### Ограничение: нельзя включить одинарную кавычку
```bash
$ echo 'It's a problem'    # Ошибка!

# Решение: выйти из кавычек
$ echo 'It'\''s fine'
It's fine

# Или использовать двойные
$ echo "It's fine"
```

## Двойные кавычки ("")

Позволяют раскрытие `$`, `$()`, обратных кавычек и `\`:

```bash
$ NAME="World"
$ echo "Hello, $NAME!"
Hello, World!

$ echo "Today is $(date)"
Today is Thu Dec 26 14:30:00 MSK 2024

$ echo "Path: $HOME"
Path: /home/user
```

### Что НЕ раскрывается
```bash
$ echo "*.txt"
*.txt                    # glob не работает

$ echo "~"
~                        # tilde не работает

$ echo "{a,b,c}"
{a,b,c}                  # brace не работает
```

### Экранирование в двойных кавычках
```bash
$ echo "Price: \$100"
Price: $100

$ echo "Line 1\nLine 2"
Line 1\nLine 2           # \n — не спецсимвол

$ echo "Say \"Hello\""
Say "Hello"
```

### Сохранение пробелов и переносов
```bash
$ OUTPUT="Hello   World"
$ echo $OUTPUT           # Hello World (пробелы сжаты)
$ echo "$OUTPUT"         # Hello   World (сохранены)
```

## Обратный слэш (\)

Экранирует один следующий символ:

```bash
$ echo \$HOME
$HOME

$ echo Hello\ World      # экранированный пробел
Hello World

$ ls file\ with\ spaces.txt

$ echo \\                # сам обратный слэш
\
```

### Продолжение строки
```bash
$ echo "This is a very \
long command that \
spans multiple lines"
```

## $'...' — ANSI-C кавычки

Позволяют использовать escape-последовательности:

```bash
$ echo $'Line 1\nLine 2'
Line 1
Line 2

$ echo $'Tab:\there'
Tab:    here

$ echo $'Quote: \''
Quote: '
```

| Sequence | Значение |
|----------|----------|
| `\n` | Новая строка |
| `\t` | Табуляция |
| `\\` | Обратный слэш |
| `\'` | Одинарная кавычка |
| `\"` | Двойная кавычка |
| `\xHH` | Hex-код символа |

## Практические примеры

### Работа с путями с пробелами
```bash
# Неправильно
$ cd /path/to/My Documents
bash: cd: too many arguments

# Правильно
$ cd "/path/to/My Documents"
$ cd '/path/to/My Documents'
$ cd /path/to/My\ Documents
```

### Передача аргументов
```bash
$ FILE="my file.txt"

# Неправильно — word splitting
$ cat $FILE
cat: my: No such file or directory
cat: file.txt: No such file or directory

# Правильно
$ cat "$FILE"
```

### Массивы в кавычках
```bash
$ FILES=("file one.txt" "file two.txt")

# Неправильно
$ echo ${FILES[*]}     # одна строка

# Правильно — каждый элемент отдельно
$ for f in "${FILES[@]}"; do echo "$f"; done
file one.txt
file two.txt
```

### Безопасная обработка ввода
```bash
#!/bin/bash
read -r INPUT
# Всегда используйте кавычки
echo "You entered: $INPUT"
```

### Сложные команды
```bash
$ ssh server "cd /app && echo 'App path: $PWD'"
# $PWD раскроется локально

$ ssh server 'cd /app && echo "App path: $PWD"'
# $PWD раскроется на сервере
```

## Когда что использовать

### Одинарные кавычки `''`
- Фиксированные строки без переменных
- Регулярные выражения
- Шаблоны для grep/sed/awk
- Когда нужен буквальный `$`

### Двойные кавычки `""`
- Строки с переменными
- Пути к файлам
- Большинство случаев в скриптах
- Когда нужны пробелы

### Без кавычек
- Числа
- Простые имена файлов без спецсимволов
- Когда нужен glob (`*.txt`)
- Когда нужен brace expansion

## Правило: "Кавычки везде!"

В скриптах безопаснее всегда использовать двойные кавычки:

```bash
#!/bin/bash
FILE="$1"                    # в кавычках
DIR="${FILE%/*}"             # в кавычках
cp "$FILE" "$DIR/backup/"    # в кавычках
echo "Done: $FILE"           # в кавычках
```

## Вложенные кавычки

```bash
$ echo "He said \"Hello\""
He said "Hello"

$ bash -c "echo 'inner quotes'"
inner quotes

$ bash -c 'echo "inner quotes"'
inner quotes
```

## Частые ошибки

```bash
# Пропущенные кавычки
$ FILE=my file.txt           # Ошибка!
$ FILE="my file.txt"         # Правильно

# Неправильные кавычки
$ echo "Path: ~"             # ~ не раскроется
$ echo "Path: $HOME"         # Правильно

# Смешение кавычек
$ echo "It's a "problem""    # Ошибка!
$ echo "It's a \"problem\""  # Правильно
```
