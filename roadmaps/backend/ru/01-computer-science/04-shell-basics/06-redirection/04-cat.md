# Команда cat

## Что такое cat?

**cat** (concatenate) — команда для вывода содержимого файлов и объединения нескольких файлов.

## Синтаксис

```bash
cat [опции] [файл...]
```

## Базовое использование

### Вывод содержимого файла
```bash
$ cat file.txt
Hello, World!
This is a text file.
```

### Вывод нескольких файлов
```bash
$ cat file1.txt file2.txt
Content of file1
Content of file2
```

### Объединение файлов
```bash
$ cat part1.txt part2.txt > combined.txt
```

## Основные опции

### -n — нумерация строк
```bash
$ cat -n file.txt
     1  First line
     2  Second line
     3  Third line
```

### -b — нумерация непустых строк
```bash
$ cat -b file.txt
     1  First line

     2  Second line
```

### -s — убрать повторяющиеся пустые строки
```bash
$ cat -s file.txt
# Множественные пустые строки заменяются одной
```

### -A — показать все (непечатаемые символы)
```bash
$ cat -A file.txt
Hello^IWorld$      # ^I = tab, $ = конец строки
```

### -E — показать конец строки ($)
```bash
$ cat -E file.txt
Hello World$
```

### -T — показать табы как ^I
```bash
$ cat -T file.txt
Name^IValue
```

### -v — показать непечатаемые символы
```bash
$ cat -v binary_file
^@^@^A^B^C...
```

## Создание файлов

### С помощью перенаправления
```bash
$ cat > newfile.txt
Type some text
Press Ctrl+D when done
^D

$ cat newfile.txt
Type some text
Press Ctrl+D when done
```

### Добавление в файл
```bash
$ cat >> file.txt
Additional content
^D
```

### Here Document
```bash
$ cat > script.sh << EOF
#!/bin/bash
echo "Hello, World!"
EOF
```

## Практические примеры

### Объединение частей файла
```bash
$ cat header.txt content.txt footer.txt > page.html
```

### Просмотр конфигурации
```bash
$ cat /etc/passwd
$ cat /etc/hosts
```

### Быстрое создание файла
```bash
$ cat > notes.txt << EOF
Meeting notes:
- Point 1
- Point 2
- Point 3
EOF
```

### Вывод с номерами строк для отладки
```bash
$ cat -n script.sh
     1  #!/bin/bash
     2  echo "Hello"
     3  exit 0
```

### Проверка скрытых символов
```bash
$ cat -A file.txt
# Полезно для поиска проблем с пробелами/табами
```

## Связь с перенаправлением

### cat как приёмник данных
```bash
$ echo "Hello" | cat
Hello

$ ls | cat > listing.txt
```

### cat как источник данных
```bash
$ cat file.txt | grep "pattern"
$ cat file.txt | wc -l
$ cat file.txt | sort | uniq
```

### cat с stdin
```bash
$ cat -              # читать из stdin
$ cat file.txt -     # файл, затем stdin
$ cat - file.txt     # stdin, затем файл
```

## Когда НЕ использовать cat

### Бесполезное использование cat (UUOC)
```bash
# Плохо (лишний процесс):
$ cat file.txt | grep "pattern"

# Лучше:
$ grep "pattern" file.txt
```

```bash
# Плохо:
$ cat file.txt | wc -l

# Лучше:
$ wc -l < file.txt
```

### Когда cat оправдан

```bash
# Объединение файлов
$ cat file1.txt file2.txt | process

# Чтение нескольких источников
$ cat header.txt - footer.txt

# Просмотр небольшого файла
$ cat config.txt
```

## Альтернативы cat

### less — для больших файлов
```bash
$ less largefile.txt
# Постраничный просмотр
```

### head / tail — начало/конец файла
```bash
$ head -10 file.txt    # первые 10 строк
$ tail -10 file.txt    # последние 10 строк
```

### tac — cat наоборот
```bash
$ tac file.txt
Third line
Second line
First line
```

### bat — cat с подсветкой синтаксиса
```bash
$ bat script.py
# Цветной вывод с номерами строк
```

### rev — перевернуть строки
```bash
$ echo "Hello" | rev
olleH
```

## Полезные комбинации

### Очистка файла (не удаляя)
```bash
$ cat /dev/null > file.txt
```

### Копирование файла
```bash
$ cat source.txt > dest.txt
# То же что cp, но медленнее
```

### Добавление заголовка к файлу
```bash
$ cat header.txt main.txt > result.txt
$ mv result.txt main.txt
```

### Создание простого бэкапа
```bash
$ cat config.conf > config.conf.bak
```

## Советы

1. **Для больших файлов** используйте `less` или `head`/`tail`
2. **Для поиска** используйте `grep file.txt` вместо `cat file.txt | grep`
3. **Для проверки формата** используйте `cat -A`
4. **Для объединения файлов** cat — идеальный инструмент
5. **Установите bat** для цветного вывода с подсветкой
