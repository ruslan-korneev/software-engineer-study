# Pathname Expansion (Globbing)

## Что такое pathname expansion?

**Pathname expansion** (раскрытие путей, также называемое globbing) — механизм shell, который преобразует шаблоны с wildcards в список подходящих файлов.

```bash
$ echo *.txt
file1.txt file2.txt notes.txt
# Shell раскрыл *.txt в список файлов
```

## Порядок раскрытия

Shell выполняет раскрытия в определённом порядке:
1. Brace expansion
2. Tilde expansion
3. Parameter expansion
4. Command substitution
5. Arithmetic expansion
6. Word splitting
7. **Pathname expansion** (globbing)

## Основные wildcards

### * — любое количество символов

```bash
$ ls *.txt           # все .txt файлы
$ ls file*           # всё начинающееся с "file"
$ ls *report*        # всё содержащее "report"
$ ls *.tar.gz        # все .tar.gz архивы
```

### ? — ровно один символ

```bash
$ ls file?.txt       # file1.txt, fileA.txt, но не file10.txt
$ ls ???.txt         # abc.txt, 123.txt (ровно 3 символа)
```

### [...] — один символ из набора

```bash
$ ls file[123].txt   # file1.txt, file2.txt, file3.txt
$ ls file[a-z].txt   # filea.txt, fileb.txt, ...
$ ls file[0-9].txt   # file0.txt, file1.txt, ...
```

### [!...] или [^...] — исключение

```bash
$ ls file[!0-9].txt  # файлы НЕ с цифрой
$ ls [^.]*.txt       # .txt файлы, НЕ начинающиеся с точки
```

## Классы символов POSIX

| Класс | Описание |
|-------|----------|
| `[[:alpha:]]` | Буквы |
| `[[:digit:]]` | Цифры |
| `[[:alnum:]]` | Буквы и цифры |
| `[[:lower:]]` | Строчные буквы |
| `[[:upper:]]` | Заглавные буквы |
| `[[:space:]]` | Пробельные символы |
| `[[:punct:]]` | Знаки пунктуации |

```bash
$ ls [[:upper:]]*.txt     # файлы, начинающиеся с заглавной
$ ls *[[:digit:]].txt     # файлы, заканчивающиеся цифрой
```

## Скрытые файлы

По умолчанию `*` не включает скрытые файлы (начинающиеся с `.`):

```bash
$ ls *                # не покажет .bashrc
$ ls .*               # только скрытые
$ ls * .*             # все файлы
```

В bash можно включить скрытые файлы в `*`:
```bash
$ shopt -s dotglob
$ ls *                # теперь включает скрытые
$ shopt -u dotglob    # вернуть обратно
```

## Рекурсивное раскрытие (**)

В bash (с `globstar`) и zsh:

```bash
$ shopt -s globstar   # включить в bash
$ ls **/*.txt         # все .txt во всех поддиректориях
$ ls **/README.md     # все README.md рекурсивно
```

## Когда нет совпадений

Если шаблон не находит файлов, поведение зависит от настроек:

```bash
$ ls *.xyz
ls: cannot access '*.xyz': No such file or directory
# Шаблон остался буквально
```

Можно изменить поведение:
```bash
$ shopt -s nullglob   # раскрыть в пустоту
$ echo *.xyz
                      # пустая строка

$ shopt -s failglob   # вызвать ошибку
$ echo *.xyz
bash: no match: *.xyz
```

## Практические примеры

### Операции с группами файлов
```bash
$ cp *.jpg images/           # все JPEG
$ rm *.tmp                   # все временные
$ mv report-*.pdf archive/   # все отчёты
```

### Фильтрация по имени
```bash
$ ls log-2024-12-??.txt      # логи за декабрь 2024
$ cat config-[dp]*.yml       # config-dev.yml, config-prod.yml
```

### Исключение файлов
```bash
$ ls !(*.txt)                # всё кроме .txt (extended globbing)
$ shopt -s extglob           # нужно включить
```

## Extended Globbing

В bash можно включить расширенные шаблоны:

```bash
$ shopt -s extglob
```

| Шаблон | Описание |
|--------|----------|
| `?(pattern)` | 0 или 1 совпадение |
| `*(pattern)` | 0 или более совпадений |
| `+(pattern)` | 1 или более совпадений |
| `@(pattern)` | Ровно 1 совпадение |
| `!(pattern)` | Всё кроме pattern |

```bash
$ ls *.@(jpg|png|gif)        # изображения
$ ls !(*.bak)                # всё кроме .bak
$ ls +([0-9]).txt            # 1.txt, 123.txt (только цифры)
```

## echo для проверки

Используйте `echo` чтобы увидеть, во что раскроется шаблон:

```bash
$ echo *.txt
file1.txt file2.txt notes.txt

$ echo rm *.log              # посмотреть что будет удалено
rm access.log error.log debug.log
```

## Экранирование

Чтобы использовать символ буквально:

```bash
$ ls file\*.txt              # файл с именем "file*.txt"
$ ls "file*.txt"             # то же
$ ls 'file?.txt'             # файл с именем "file?.txt"
```

## Важные замечания

1. **Раскрытие происходит в shell**, программа получает готовый список
2. **Порядок файлов** — алфавитный (locale-зависимый)
3. **Пробелы в именах** могут вызвать проблемы — используйте кавычки
4. **`*` не пересекает `/`** — для рекурсии нужен `**`

```bash
$ ls */*.txt         # только один уровень вложенности
$ ls **/*.txt        # все уровни (с globstar)
```
