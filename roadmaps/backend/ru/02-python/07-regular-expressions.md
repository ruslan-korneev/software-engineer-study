# Регулярные выражения (Regular Expressions)

## Что это?

**Regex** — язык для поиска и обработки текста по шаблонам.

```python
import re

text = "Email: test@example.com"
match = re.search(r'\S+@\S+', text)
match.group()  # 'test@example.com'
```

**Важно:** используй `r'...'` (raw string) для паттернов.

## Модуль re — основные функции

| Функция | Описание |
|---------|----------|
| `re.search(pattern, string)` | Первое совпадение |
| `re.match(pattern, string)` | Совпадение в начале строки |
| `re.findall(pattern, string)` | Список всех совпадений |
| `re.finditer(pattern, string)` | Итератор Match-объектов |
| `re.sub(pattern, repl, string)` | Замена |
| `re.split(pattern, string)` | Разбить по паттерну |
| `re.compile(pattern)` | Скомпилировать |

## Метасимволы

| Символ | Значение |
|--------|----------|
| `.` | Любой символ (кроме \n) |
| `^` | Начало строки |
| `$` | Конец строки |
| `\|` | ИЛИ |
| `\` | Экранирование |

```python
re.search(r'^Hello', 'Hello world')  # Match
re.search(r'world$', 'Hello world')  # Match
re.search(r'cat|dog', 'I have a cat')  # Match
```

## Классы символов

| Паттерн | Значение |
|---------|----------|
| `\d` | Цифра [0-9] |
| `\D` | НЕ цифра |
| `\w` | Буква, цифра, _ |
| `\W` | НЕ \w |
| `\s` | Пробельный символ |
| `\S` | НЕ пробельный |

### Свои классы [...]

```python
[abc]      # a, b или c
[a-z]      # любая буква a-z
[0-9]      # цифра
[^abc]     # НЕ a, b, c (отрицание)
```

## Квантификаторы

| Квантификатор | Значение |
|---------------|----------|
| `*` | 0 или более |
| `+` | 1 или более |
| `?` | 0 или 1 |
| `{n}` | Ровно n |
| `{n,}` | n или более |
| `{n,m}` | От n до m |

```python
re.findall(r'\d+', 'a1b22c333')    # ['1', '22', '333']
re.findall(r'\d{2,}', 'a1b22c333') # ['22', '333']
```

### Жадные vs ленивые

```python
text = '<div>content</div>'
re.search(r'<.*>', text).group()   # '<div>content</div>' — жадный
re.search(r'<.*?>', text).group()  # '<div>' — ленивый
```

Добавь `?` для ленивого режима: `*?`, `+?`, `??`

## Группы

### Захватывающие (...)

```python
text = '2024-01-15'
match = re.search(r'(\d{4})-(\d{2})-(\d{2})', text)

match.group()   # '2024-01-15'
match.group(1)  # '2024'
match.groups()  # ('2024', '01', '15')
```

### Именованные (?P<name>...)

```python
match = re.search(r'(?P<year>\d{4})-(?P<month>\d{2})', '2024-01')

match.group('year')  # '2024'
match.groupdict()    # {'year': '2024', 'month': '01'}
```

### Незахватывающие (?:...)

```python
re.findall(r'(?:https?)://(\S+)', 'http://example.com')
# ['example.com'] — протокол не захвачен
```

## Функции подробнее

### search vs match

```python
text = 'Hello World'
re.match(r'World', text)   # None — только с начала
re.search(r'World', text)  # Match — везде
```

### findall

```python
re.findall(r'[cbr]at', 'cat bat rat')  # ['cat', 'bat', 'rat']

# С группами возвращает только группы
re.findall(r'(\w+)@(\w+)', 'a@b c@d')  # [('a', 'b'), ('c', 'd')]
```

### sub — замена

```python
re.sub(r'World', 'Python', 'Hello World')  # 'Hello Python'

# С группами — обратные ссылки \1, \2
re.sub(r'(\d{4})-(\d{2})-(\d{2})', r'\3/\2/\1', '2024-01-15')
# '15/01/2024'

# С функцией
re.sub(r'\w+', lambda m: m.group().upper(), 'hello')  # 'HELLO'
```

### split

```python
re.split(r'\s+', 'hello   world')  # ['hello', 'world']
re.split(r'[,;]\s*', 'a, b; c')    # ['a', 'b', 'c']
```

### compile

```python
email_re = re.compile(r'\S+@\S+\.\S+')
email_re.findall('a@b.com and c@d.org')  # ['a@b.com', 'c@d.org']
```

## Флаги

| Флаг | Значение |
|------|----------|
| `re.I` | Без учёта регистра |
| `re.M` | Multiline: `^`/`$` для каждой строки |
| `re.S` | `.` включает `\n` |
| `re.X` | Verbose: пробелы и комментарии |

```python
re.search(r'hello', 'HELLO', re.I)  # Match

pattern = re.compile(r'''
    \d{4}    # год
    -\d{2}   # месяц
    -\d{2}   # день
''', re.X)
```

## Частые паттерны

```python
# Email
r'[\w.-]+@[\w.-]+\.\w+'

# URL
r'https?://\S+'

# IP адрес
r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}'

# Телефон
r'\+?[\d\s\-\(\)]+'

# HTML тег
r'<[^>]+>'
```

## Q&A

**Q: Зачем r'...' перед паттерном?**
A: Raw string — отключает экранирование Python. Без него `\d` нужно писать как `\\d`.

**Q: Разница между search и match?**
A: `match` — только с начала строки, `search` — ищет везде.

**Q: Что значит жадный/ленивый?**
A: Жадный (`*`, `+`) захватывает максимум символов. Ленивый (`*?`, `+?`) — минимум.

**Q: Когда использовать compile?**
A: Когда один паттерн используется много раз — экономит время на компиляции.
