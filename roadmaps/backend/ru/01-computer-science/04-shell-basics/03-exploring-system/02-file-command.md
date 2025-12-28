# Команда file

[prev: 01-ls-options](./01-ls-options.md) | [next: 03-less-command](./03-less-command.md)

---
## Что такое file?

**file** — команда для определения типа файла. В отличие от Windows, где тип файла определяется расширением, в Unix/Linux тип определяется по содержимому файла.

## Синтаксис

```bash
file [опции] файл...
```

## Базовое использование

```bash
$ file document.pdf
document.pdf: PDF document, version 1.4

$ file photo.jpg
photo.jpg: JPEG image data, JFIF standard 1.01

$ file script.sh
script.sh: Bourne-Again shell script, ASCII text executable
```

## Почему это важно?

В Unix файл может не иметь расширения или иметь неправильное:

```bash
$ file mystery_file
mystery_file: PNG image data, 800 x 600, 8-bit/color RGBA

# Файл без расширения — на самом деле PNG изображение!
```

Также расширение может быть обманчивым:

```bash
$ file virus.jpg
virus.jpg: ELF 64-bit executable, x86-64

# Это не картинка, а исполняемый файл!
```

## Типы файлов

### Текстовые файлы
```bash
$ file README.md
README.md: UTF-8 Unicode text

$ file config.json
config.json: JSON data

$ file script.py
script.py: Python script, ASCII text executable
```

### Исполняемые файлы
```bash
$ file /bin/ls
/bin/ls: ELF 64-bit LSB executable, x86-64

$ file /usr/bin/python3
/usr/bin/python3: symbolic link to python3.10
```

### Архивы
```bash
$ file backup.tar.gz
backup.tar.gz: gzip compressed data

$ file archive.zip
archive.zip: Zip archive data
```

### Мультимедиа
```bash
$ file video.mp4
video.mp4: ISO Media, MP4 v2

$ file song.mp3
song.mp3: Audio file with ID3 version 2.4.0
```

### Директории и ссылки
```bash
$ file /home
/home: directory

$ file /usr/bin/python
/usr/bin/python: symbolic link to python3
```

## Полезные опции

### -i — MIME-тип
```bash
$ file -i document.pdf
document.pdf: application/pdf; charset=binary

$ file -i index.html
index.html: text/html; charset=utf-8
```

### -b — краткий вывод (без имени файла)
```bash
$ file -b photo.jpg
JPEG image data, JFIF standard 1.01
```

### -L — следовать по символическим ссылкам
```bash
$ file /usr/bin/python
/usr/bin/python: symbolic link to python3

$ file -L /usr/bin/python
/usr/bin/python: ELF 64-bit LSB executable
```

### -z — заглянуть внутрь сжатых файлов
```bash
$ file -z backup.tar.gz
backup.tar.gz: POSIX tar archive (gzip compressed data)
```

## Практические примеры

### Проверка нескольких файлов
```bash
$ file *
README.md:    UTF-8 Unicode text
config.json:  JSON data
script.sh:    Bourne-Again shell script
image.png:    PNG image data
```

### Поиск определённых типов файлов
```bash
# Найти все текстовые файлы
$ file * | grep text
README.md:    UTF-8 Unicode text
config.yaml:  ASCII text

# Найти все изображения
$ file * | grep image
photo.jpg:    JPEG image data
icon.png:     PNG image data
```

### Проверка бинарных файлов
```bash
$ file /bin/*
/bin/bash:  ELF 64-bit LSB executable, x86-64
/bin/cat:   ELF 64-bit LSB executable, x86-64
/bin/ls:    ELF 64-bit LSB executable, x86-64
```

### Определение кодировки текста
```bash
$ file document.txt
document.txt: UTF-8 Unicode text

$ file old_document.txt
old_document.txt: ISO-8859 text
```

## Как file определяет тип?

Команда `file` использует базу данных "magic numbers" — специальных байтов в начале файла:

| Тип файла | Magic bytes |
|-----------|-------------|
| PNG | `89 50 4E 47` |
| JPEG | `FF D8 FF` |
| PDF | `%PDF` |
| ZIP | `PK` |
| ELF | `7F 45 4C 46` |

База данных находится обычно в `/usr/share/file/magic`.

## Использование в скриптах

```bash
#!/bin/bash

# Проверить, является ли файл изображением
if file "$1" | grep -q "image"; then
    echo "Это изображение"
else
    echo "Это не изображение"
fi
```

## Советы

1. Всегда проверяйте файлы из ненадёжных источников командой `file`
2. Не доверяйте расширениям файлов — используйте `file` для проверки
3. Используйте `file -i` для получения MIME-типа (полезно для веб-разработки)

---

[prev: 01-ls-options](./01-ls-options.md) | [next: 03-less-command](./03-less-command.md)
