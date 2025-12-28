# Wildcards (подстановочные символы)

[prev: 04-symbolic-hard-links](../03-exploring-system/04-symbolic-hard-links.md) | [next: 02-mkdir](./02-mkdir.md)

---
## Что такое wildcards?

**Wildcards** (подстановочные символы, шаблоны) — это специальные символы, которые позволяют выбирать несколько файлов одновременно по шаблону. Shell раскрывает эти шаблоны в список соответствующих файлов.

## Основные wildcards

### * (звёздочка) — любое количество символов

Соответствует любой последовательности символов (включая пустую):

```bash
$ ls *.txt           # все файлы .txt
file.txt  notes.txt  readme.txt

$ ls image*          # всё, начинающееся с "image"
image.jpg  image.png  image2.jpg  images/

$ ls *report*        # всё, содержащее "report"
report.pdf  annual-report.doc  reports/
```

### ? (вопросительный знак) — ровно один символ

Соответствует любому одному символу:

```bash
$ ls file?.txt       # file + один символ + .txt
file1.txt  file2.txt  fileA.txt

$ ls ???.txt         # ровно 3 символа + .txt
abc.txt  log.txt  123.txt

$ ls image?.jpg
image1.jpg  image2.jpg  imageA.jpg
```

### [...] (квадратные скобки) — один символ из набора

Соответствует любому символу из указанного набора:

```bash
$ ls file[123].txt   # file1.txt, file2.txt или file3.txt
file1.txt  file2.txt

$ ls file[abc].txt   # filea.txt, fileb.txt или filec.txt
filea.txt  filec.txt
```

### [a-z] — диапазон символов

```bash
$ ls file[a-z].txt   # file + любая маленькая буква + .txt
filea.txt  fileb.txt  filez.txt

$ ls file[0-9].txt   # file + цифра + .txt
file0.txt  file5.txt  file9.txt

$ ls file[A-Za-z].txt # file + любая буква + .txt
fileA.txt  filea.txt  fileZ.txt
```

### [!...] или [^...] — исключение символов

Соответствует любому символу, кроме указанных:

```bash
$ ls file[!0-9].txt  # file + НЕ цифра + .txt
filea.txt  fileX.txt

$ ls file[^abc].txt  # file + НЕ a, b, c + .txt
filed.txt  file1.txt
```

## Классы символов POSIX

Для переносимости между системами используют классы:

| Класс | Описание |
|-------|----------|
| `[[:alpha:]]` | Любая буква |
| `[[:digit:]]` | Любая цифра |
| `[[:alnum:]]` | Буква или цифра |
| `[[:lower:]]` | Строчная буква |
| `[[:upper:]]` | Заглавная буква |
| `[[:space:]]` | Пробельный символ |

```bash
$ ls file[[:digit:]].txt
file1.txt  file9.txt

$ ls [[:upper:]]*.txt     # начинается с заглавной
Document.txt  README.txt
```

## Практические примеры

### Операции с группами файлов
```bash
# Скопировать все изображения
$ cp *.jpg *.png images/

# Переместить все логи
$ mv *.log logs/

# Удалить все временные файлы
$ rm *.tmp

# Показать все скрипты
$ ls *.sh *.py *.rb
```

### Работа с расширениями
```bash
# Все файлы с двойным расширением
$ ls *.tar.gz

# Все исходные файлы
$ ls *.{c,h,cpp,hpp}
```

### Фильтрация по именам
```bash
# Файлы backup с датой
$ ls backup-????-??-??.tar
backup-2024-12-25.tar  backup-2024-12-26.tar

# Логи за декабрь
$ ls log-2024-12-*.txt
```

## Важные особенности

### Скрытые файлы
По умолчанию `*` не включает скрытые файлы (начинающиеся с `.`):

```bash
$ ls *           # не покажет .bashrc, .config
$ ls .*          # только скрытые
$ ls * .*        # все файлы
```

### Wildcard не находит совпадений
Если нет совпадений, shell оставляет шаблон как есть:

```bash
$ ls *.xyz
ls: cannot access '*.xyz': No such file or directory
```

### Раскрытие происходит в shell
Команда получает уже готовый список файлов:

```bash
$ echo *.txt
file1.txt file2.txt file3.txt
# echo получает 3 аргумента, а не "*.txt"
```

### Экранирование wildcards
Чтобы использовать символ буквально:

```bash
$ ls file\*.txt      # ищет файл с именем "file*.txt"
$ ls "file*.txt"     # то же самое
$ ls 'file?.txt'     # ищет файл с именем "file?.txt"
```

## Сравнение с регулярными выражениями

| Wildcard | Regex | Значение |
|----------|-------|----------|
| `*` | `.*` | Любое количество символов |
| `?` | `.` | Один символ |
| `[abc]` | `[abc]` | Один из набора |
| `[!abc]` | `[^abc]` | Не из набора |

Wildcards проще, но менее мощные, чем регулярные выражения.

## Советы

1. Используйте `echo` для проверки шаблона перед выполнением:
   ```bash
   $ echo rm *.log    # увидите, что будет удалено
   rm access.log error.log debug.log
   ```

2. Будьте осторожны с `rm *` — сначала проверьте!

3. Используйте `-i` для подтверждения:
   ```bash
   $ rm -i *.txt    # спросит подтверждение для каждого файла
   ```

---

[prev: 04-symbolic-hard-links](../03-exploring-system/04-symbolic-hard-links.md) | [next: 02-mkdir](./02-mkdir.md)
