# groff - форматирование документов

## Что такое groff?

**groff** (GNU troff) - система форматирования текста для создания документов. Используется для:

- Создания man-страниц
- Форматирования документов для печати
- Генерации PostScript и PDF

groff - это GNU реализация классической системы troff из UNIX.

## История и контекст

```
roff (1964) -> nroff/troff (1970s) -> groff (1990)
```

- **roff** - первый форматировщик
- **nroff** - для терминалов
- **troff** - для типографского вывода
- **groff** - современная GNU версия

## Основные макропакеты

| Пакет | Назначение |
|-------|------------|
| `man` | Man-страницы |
| `ms` | Документы общего назначения |
| `mm` | Меморандумы и отчёты |
| `me` | Технические документы |
| `mom` | Современный универсальный |

## Формат man-страниц

### Структура man-страницы

```groff
.TH COMMAND 1 "January 2024" "Version 1.0" "User Commands"
.SH NAME
command \- short description
.SH SYNOPSIS
.B command
[\fB\-option\fR]
.IR file ...
.SH DESCRIPTION
Long description of the command.
.SH OPTIONS
.TP
.BR \-h ", " \-\-help
Display help message.
.TP
.BR \-v ", " \-\-version
Display version.
.SH EXAMPLES
.PP
Example usage:
.RS
command -v file.txt
.RE
.SH SEE ALSO
.BR related (1),
.BR another (5)
.SH AUTHOR
Author Name <email@example.com>
```

### Базовые команды man

```groff
.TH title section date source manual
    - заголовок страницы

.SH section_name
    - начало секции

.SS subsection_name
    - подсекция

.PP
    - новый абзац

.TP
    - tagged paragraph (для списков опций)

.IP text
    - indented paragraph

.RS / .RE
    - относительный отступ (start/end)

.B text
    - жирный

.I text
    - курсив

.BI bold italic
    - чередование стилей

.BR text
    - жирный, затем обычный

\fB bold \fR normal \fI italic
    - переключение шрифтов inline
```

### Создание man-страницы

```bash
# Создать файл mycommand.1
cat > mycommand.1 << 'EOF'
.TH MYCOMMAND 1 "January 2024" "1.0" "User Commands"
.SH NAME
mycommand \- do something useful
.SH SYNOPSIS
.B mycommand
[\fB\-v\fR]
[\fB\-o\fR \fIoutput\fR]
.IR file ...
.SH DESCRIPTION
.B mycommand
processes files and produces output.
.SH OPTIONS
.TP
.BR \-v ", " \-\-verbose
Enable verbose mode.
.TP
.BR \-o ", " \-\-output =\fIFILE\fR
Write output to FILE.
.SH EXAMPLES
Process a file:
.PP
.RS
mycommand input.txt
.RE
.SH EXIT STATUS
.TP
.B 0
Success.
.TP
.B 1
Error occurred.
.SH AUTHOR
Written by Author Name.
EOF

# Просмотреть
man ./mycommand.1

# Установить
sudo cp mycommand.1 /usr/local/share/man/man1/
sudo mandb
```

## Работа с groff

### Базовое использование

```bash
# Форматировать man-страницу
groff -man -Tutf8 mycommand.1 | less

# Создать PostScript
groff -man -Tps mycommand.1 > output.ps

# Создать PDF
groff -man -Tpdf mycommand.1 > output.pdf
# или
groff -man -Tps mycommand.1 | ps2pdf - output.pdf

# HTML вывод
groff -man -Thtml mycommand.1 > output.html
```

### Форматы вывода (-T)

```bash
groff -Tascii   # ASCII текст
groff -Tutf8    # UTF-8 текст
groff -Tps      # PostScript
groff -Tpdf     # PDF (если поддерживается)
groff -Thtml    # HTML
groff -Tdvi     # DVI
```

## Макропакет ms

### Структура документа ms

```groff
.TL
Document Title
.AU
Author Name
.AI
Institution
.AB
Abstract text goes here.
.AE
.NH
First Section
.PP
First paragraph of the first section.
.NH 2
Subsection
.PP
Subsection content.
.SH
Unnumbered Section
.PP
Content here.
```

### Команды ms

```groff
.TL - title
.AU - author
.AI - institution
.AB / .AE - abstract begin/end
.NH n - numbered heading level n
.SH - section heading (unnumbered)
.PP - paragraph
.LP - left-aligned paragraph
.QP - quoted paragraph
.IP marker - indented paragraph
.RS / .RE - relative indent
.DS / .DE - display start/end
```

### Форматирование текста

```groff
.B bold text
.I italic text
.R roman text

\fBbold\fR normal \fIitalic\fP

.UL underline text

.BX boxed text
```

### Таблицы (tbl)

```groff
.TS
center box;
c c c
l n n.
Name	Score	Grade
_
Alice	92	A
Bob	78	C
Charlie	85	B
.TE
```

### Использование ms

```bash
# Создать документ
cat > document.ms << 'EOF'
.TL
My Document
.AU
John Doe
.PP
This is a sample document created with groff ms macros.
.NH
Introduction
.PP
Here is the introduction text.
.NH 2
Background
.PP
Some background information.
EOF

# Обработать
groff -ms -Tpdf document.ms > document.pdf
```

## Предпроцессоры

### tbl - таблицы

```bash
# Файл с таблицей
cat > table.ms << 'EOF'
.TS
allbox;
c c c
l n n.
Item	Qty	Price
Apple	10	$1.00
Banana	15	$0.50
.TE
EOF

# Обработка
tbl table.ms | groff -ms -Tpdf > table.pdf
# или
groff -t -ms -Tpdf table.ms > table.pdf
```

### eqn - формулы

```bash
# Файл с формулой
cat > equation.ms << 'EOF'
.EQ
x = {-b +- sqrt{b sup 2 - 4ac}} over 2a
.EN
EOF

# Обработка
eqn equation.ms | groff -ms -Tpdf > equation.pdf
# или
groff -e -ms -Tpdf equation.ms > equation.pdf
```

### pic - диаграммы

```bash
# Файл с диаграммой
cat > diagram.ms << 'EOF'
.PS
box "Start"
arrow
box "Process"
arrow
box "End"
.PE
EOF

# Обработка
groff -p -ms -Tpdf diagram.ms > diagram.pdf
```

## Полезные команды

### man2html

```bash
# Конвертировать man в HTML
man2html < command.1 > command.html

# Или через groff
groff -man -Thtml command.1 > command.html
```

### Просмотр man-страниц

```bash
# Стандартный просмотр
man command

# Из файла
man ./mycommand.1

# В определённой секции
man 5 passwd    # passwd(5)

# Все секции
man -a command

# Поиск
man -k keyword
apropos keyword
```

### Проверка man-страницы

```bash
# Проверить синтаксис
mandoc -Tlint mycommand.1

# Форматирование проверки
groff -man -ww -z mycommand.1
```

## Альтернативы groff

### Pandoc

```bash
# Markdown в man
pandoc -s -t man README.md -o command.1

# man в Markdown
pandoc -s command.1 -o README.md
```

### ronn

```bash
# Установка
gem install ronn

# Markdown в man
ronn command.1.md
```

### scdoc

```bash
# Установка
sudo apt install scdoc

# Простой формат в man
scdoc < command.scd > command.1
```

### help2man

```bash
# Автогенерация man из --help
help2man ./myprogram > myprogram.1
```

## Резюме

### Команды groff

| Команда | Описание |
|---------|----------|
| `groff -man file.1` | Форматировать man-страницу |
| `groff -ms file.ms` | Использовать макросы ms |
| `groff -Tpdf` | Вывод в PDF |
| `groff -Thtml` | Вывод в HTML |
| `groff -t` | Предобработка таблиц |
| `groff -e` | Предобработка уравнений |
| `groff -p` | Предобработка диаграмм |

### Секции man

| Секция | Содержимое |
|--------|------------|
| 1 | Пользовательские команды |
| 2 | Системные вызовы |
| 3 | Библиотечные функции |
| 4 | Специальные файлы |
| 5 | Форматы файлов |
| 6 | Игры |
| 7 | Разное |
| 8 | Администрирование |
