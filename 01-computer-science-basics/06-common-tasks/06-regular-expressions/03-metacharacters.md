# Метасимволы в регулярных выражениях

## Что такое метасимволы?

**Метасимволы** - это специальные символы, которые имеют особое значение в регулярных выражениях. Они не совпадают сами с собой, а определяют правила поиска.

## Основные метасимволы

### Точка (.) - любой символ

```bash
# Точка соответствует любому одному символу (кроме \n)
grep 'h.t' words.txt
# hat, hit, hot, h@t, h1t...

grep 'c..t' words.txt
# cart, coat, cost, c00t...

# Три любых символа
grep '...' words.txt
# любые слова из 3+ символов
```

### Якоря: ^ и $

```bash
# ^ - начало строки
grep '^Error' log.txt
# Error: something went wrong
# Errorcode: 500

# $ - конец строки
grep 'done$' log.txt
# Task done
# Process done

# Комбинация - точное совпадение строки
grep '^exact match$' file.txt

# Пустые строки
grep '^$' file.txt
```

### Экранирование (\\)

Обратный слеш превращает метасимвол в обычный символ:

```bash
# Искать точку как символ
grep 'file\.txt' files.txt

# Искать звёздочку
grep '\*\*\*' reviews.txt

# Искать знак доллара
grep '\$100' prices.txt

# Искать обратный слеш
grep '\\' paths.txt
```

## Классы символов [...]

### Базовые классы

```bash
# Один символ из набора
grep '[aeiou]' words.txt      # любая гласная
grep '[abc]' text.txt         # a или b или c

# Диапазоны
grep '[a-z]' text.txt         # любая строчная буква
grep '[A-Z]' text.txt         # любая заглавная буква
grep '[0-9]' text.txt         # любая цифра
grep '[a-zA-Z]' text.txt      # любая буква
grep '[a-zA-Z0-9]' text.txt   # буква или цифра

# Комбинация
grep '[aeiouAEIOU]' text.txt  # любая гласная
```

### Отрицание [^...]

```bash
# НЕ символ из набора
grep '[^0-9]' text.txt        # не цифра
grep '[^a-z]' text.txt        # не строчная буква
grep '[^aeiou]' text.txt      # не гласная (согласные + спецсимволы)

# Первый символ - не цифра
grep '^[^0-9]' lines.txt
```

### Специальные символы в классах

```bash
# Дефис в начале или конце
grep '[-abc]' text.txt        # дефис или a или b или c
grep '[abc-]' text.txt        # то же самое

# Закрывающая скобка в начале
grep '[]abc]' text.txt        # ] или a или b или c

# Карет не в начале
grep '[a^bc]' text.txt        # a или ^ или b или c

# Точка в классе - обычный символ
grep '[.]' text.txt           # только точка
```

## Квантификаторы

### Звёздочка (*) - ноль или более

```bash
# Любое количество предыдущего символа (включая 0)
grep 'go*d' words.txt
# gd, god, good, goood...

grep 'ab*c' text.txt
# ac, abc, abbc, abbbc...

# .* - любые символы (любое количество)
grep 'start.*end' text.txt
# startend, start123end, start anything end...
```

### Плюс (+) - один или более

```bash
# ERE или экранирование в BRE
grep -E 'go+d' words.txt
grep 'go\+d' words.txt
# god, good, goood... (но НЕ gd)

grep -E 'ab+c' text.txt
# abc, abbc, abbbc... (но НЕ ac)
```

### Вопрос (?) - ноль или один

```bash
# Опциональный символ
grep -E 'colou?r' text.txt
# color, colour

grep -E 'https?' urls.txt
# http, https

grep -E 'files?' text.txt
# file, files
```

### Фигурные скобки {n,m} - точное количество

```bash
# Точно n раз
grep -E 'a{3}' text.txt       # aaa

# От n до m раз
grep -E 'a{2,4}' text.txt     # aa, aaa, aaaa

# Минимум n раз
grep -E 'a{2,}' text.txt      # aa, aaa, aaaa, aaaaa...

# Максимум m раз
grep -E 'a{,3}' text.txt      # (пусто), a, aa, aaa

# BRE - нужно экранирование
grep 'a\{3\}' text.txt
grep 'a\{2,4\}' text.txt
```

## Альтернатива (|)

```bash
# Или одно, или другое (только ERE)
grep -E 'cat|dog' animals.txt
grep -E 'error|warning|critical' log.txt

# Группировка альтернатив
grep -E '(cat|dog)s?' animals.txt   # cat, cats, dog, dogs

# Несколько альтернатив
grep -E 'red|green|blue' colors.txt
```

## Группировка (...)

```bash
# Группировать для квантификатора
grep -E '(ab)+' text.txt      # ab, abab, ababab...

# Альтернативы внутри паттерна
grep -E 'file(name)?\.txt' files.txt  # file.txt, filename.txt

# Сложные паттерны
grep -E '(https?://)?www\.' urls.txt

# BRE - экранирование
grep '\(ab\)\+' text.txt
```

## Обратные ссылки

```bash
# \1 ссылается на первую группу
grep -E '(.)\1' text.txt      # любой символ, повторённый дважды (aa, bb, 11)

# Найти одинаковые слова подряд
grep -E '\b(\w+)\s+\1\b' text.txt  # "the the", "is is"

# В BRE
grep '\(.\)\1' text.txt

# Несколько групп
grep -E '(.)(.)\2\1' text.txt   # abba-паттерн
```

## Границы слов

### PCRE (grep -P)

```bash
# \b - граница слова
grep -P '\berror\b' log.txt   # только "error", не "errors"

# \B - НЕ граница слова
grep -P 'error\B' log.txt     # "errors" но не "error"
```

### GNU grep

```bash
# \< - начало слова
grep '\<error' log.txt

# \> - конец слова
grep 'error\>' log.txt

# Комбинация
grep '\<error\>' log.txt      # только целое слово

# Или просто -w
grep -w 'error' log.txt
```

## Практические примеры

### Валидация данных

```bash
# Email (упрощённый)
grep -E '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' emails.txt

# IP адрес (упрощённый)
grep -E '^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$' ips.txt

# Телефон (разные форматы)
grep -E '^\+?[0-9]{1,3}[-. ]?[0-9]{3}[-. ]?[0-9]{4}$' phones.txt

# Дата YYYY-MM-DD
grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' dates.txt
```

### Поиск в логах

```bash
# Строки с ошибками
grep -E '(ERROR|CRITICAL|FATAL)' app.log

# Время и ошибка
grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}.*ERROR' app.log

# HTTP коды ошибок
grep -E ' [45][0-9]{2} ' access.log
```

### Обработка текста

```bash
# Строки из только цифр
grep -E '^[0-9]+$' data.txt

# Пустые или пробельные строки
grep -E '^[[:space:]]*$' file.txt

# Строки с минимум 5 слов
grep -E '(\b\w+\b.*){5,}' text.txt
```

## Таблица метасимволов

| Символ | Значение | Пример |
|--------|----------|--------|
| `.` | Любой символ | `a.c` = abc, a1c |
| `^` | Начало строки | `^hello` |
| `$` | Конец строки | `world$` |
| `*` | 0 или более | `ab*c` = ac, abc, abbc |
| `+` | 1 или более | `ab+c` = abc, abbc |
| `?` | 0 или 1 | `ab?c` = ac, abc |
| `{n}` | Ровно n раз | `a{3}` = aaa |
| `{n,m}` | От n до m раз | `a{2,4}` = aa, aaa, aaaa |
| `[...]` | Класс символов | `[aeiou]` |
| `[^...]` | Отрицание класса | `[^0-9]` |
| `\|` | Альтернатива | `cat\|dog` |
| `(...)` | Группа | `(ab)+` |
| `\` | Экранирование | `\.` = точка |
| `\1` | Обратная ссылка | `(.)\1` = aa |

## Резюме

### BRE vs ERE

| BRE | ERE | Значение |
|-----|-----|----------|
| `\+` | `+` | Один или более |
| `\?` | `?` | Ноль или один |
| `\{n,m\}` | `{n,m}` | Количество |
| `\(\)` | `()` | Группа |
| не поддерж. | `\|` | Альтернатива |
