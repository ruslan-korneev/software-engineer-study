# gzip и bzip2 - сжатие файлов

[prev: 04-xargs](../04-searching-files/04-xargs.md) | [next: 02-tar](./02-tar.md)

---
## Обзор утилит сжатия

В Linux существует несколько утилит для сжатия файлов:

| Утилита | Расширение | Скорость | Сжатие | Использование |
|---------|------------|----------|--------|---------------|
| gzip | .gz | Быстрая | Хорошее | Самое популярное |
| bzip2 | .bz2 | Средняя | Лучше gzip | Для лучшего сжатия |
| xz | .xz | Медленная | Отличное | Максимальное сжатие |
| lz4 | .lz4 | Очень быстрая | Умеренное | Для скорости |
| zstd | .zst | Быстрая | Отличное | Современный стандарт |

## gzip - GNU Zip

### Что такое gzip?

**gzip** (GNU zip) - самая распространённая утилита сжатия в Unix/Linux. Использует алгоритм Deflate (LZ77 + Huffman coding).

### Базовое использование

```bash
# Сжать файл (оригинал удаляется)
gzip file.txt
# Результат: file.txt.gz

# Распаковать
gzip -d file.txt.gz
# или
gunzip file.txt.gz
# Результат: file.txt

# Сохранить оригинал
gzip -k file.txt
gzip --keep file.txt
# Результат: file.txt и file.txt.gz
```

### Уровни сжатия

```bash
# -1 до -9: баланс скорость/сжатие
gzip -1 file.txt    # быстро, слабое сжатие
gzip -9 file.txt    # медленно, лучшее сжатие

# По умолчанию -6
gzip file.txt

# Максимальное сжатие
gzip --best file.txt    # эквивалент -9

# Максимальная скорость
gzip --fast file.txt    # эквивалент -1
```

### Работа с несколькими файлами

```bash
# Сжать несколько файлов (каждый отдельно)
gzip file1.txt file2.txt file3.txt
# Результат: file1.txt.gz file2.txt.gz file3.txt.gz

# Сжать все .txt файлы
gzip *.txt

# Рекурсивно в директории
gzip -r directory/
```

### Просмотр без распаковки

```bash
# Просмотреть содержимое
zcat file.txt.gz
gzip -dc file.txt.gz

# Постраничный просмотр
zless file.txt.gz
zmore file.txt.gz

# Поиск в сжатом файле
zgrep "pattern" file.txt.gz

# Сравнение сжатых файлов
zdiff file1.txt.gz file2.txt.gz
```

### Информация о сжатии

```bash
# Показать информацию
gzip -l file.txt.gz
# compressed  uncompressed  ratio  uncompressed_name
#      12345         50000  75.3%  file.txt

# Подробная информация
gzip -lv file.txt.gz

# Проверить целостность
gzip -t file.txt.gz
gzip --test file.txt.gz
```

### Работа с потоками

```bash
# Сжатие из stdin в stdout
cat file.txt | gzip > file.txt.gz

# Распаковка в stdout
gzip -dc file.txt.gz > file.txt

# В конвейере
some_command | gzip > output.gz
gzip -dc input.gz | another_command
```

### Опции gzip

| Опция | Описание |
|-------|----------|
| `-d` | Распаковать |
| `-k` | Сохранить оригинал |
| `-c` | Вывод в stdout |
| `-f` | Принудительно перезаписать |
| `-r` | Рекурсивно |
| `-t` | Проверить целостность |
| `-l` | Показать информацию |
| `-v` | Подробный вывод |
| `-1...-9` | Уровень сжатия |
| `-n` | Не сохранять имя и время |

## bzip2 - лучшее сжатие

### Что такое bzip2?

**bzip2** использует алгоритм Burrows-Wheeler, который обеспечивает лучшее сжатие чем gzip, но работает медленнее.

### Базовое использование

```bash
# Сжать файл
bzip2 file.txt
# Результат: file.txt.bz2

# Распаковать
bzip2 -d file.txt.bz2
# или
bunzip2 file.txt.bz2

# Сохранить оригинал
bzip2 -k file.txt
```

### Уровни сжатия

```bash
# -1 до -9 (по умолчанию -9)
bzip2 -1 file.txt    # быстрее, меньше памяти
bzip2 -9 file.txt    # лучшее сжатие (по умолчанию)

# Влияет на размер блока (100k - 900k)
# -1 = 100k, -9 = 900k
```

### Просмотр без распаковки

```bash
# Просмотреть содержимое
bzcat file.txt.bz2
bzip2 -dc file.txt.bz2

# Поиск
bzgrep "pattern" file.txt.bz2

# Сравнение
bzdiff file1.txt.bz2 file2.txt.bz2
```

### Проверка и восстановление

```bash
# Проверить целостность
bzip2 -t file.txt.bz2

# Восстановить повреждённый файл
bzip2recover file.txt.bz2
# Создаёт rec00001file.txt.bz2, rec00002file.txt.bz2...
```

### Опции bzip2

| Опция | Описание |
|-------|----------|
| `-d` | Распаковать |
| `-k` | Сохранить оригинал |
| `-c` | Вывод в stdout |
| `-f` | Принудительно |
| `-t` | Проверить целостность |
| `-v` | Подробный вывод |
| `-1...-9` | Размер блока |
| `-z` | Сжать (по умолчанию) |

## xz - максимальное сжатие

### Что такое xz?

**xz** использует алгоритм LZMA2, обеспечивая лучшее сжатие среди стандартных утилит, но требует больше времени и памяти.

### Базовое использование

```bash
# Сжать файл
xz file.txt
# Результат: file.txt.xz

# Распаковать
xz -d file.txt.xz
# или
unxz file.txt.xz

# Сохранить оригинал
xz -k file.txt
```

### Уровни сжатия

```bash
# -0 до -9 (по умолчанию -6)
xz -0 file.txt    # быстро
xz -9 file.txt    # максимум

# Экстремальное сжатие
xz -9e file.txt
xz --extreme file.txt
```

### Многопоточность

```bash
# Использовать все ядра
xz -T 0 file.txt
xz --threads=0 file.txt

# Указать количество потоков
xz -T 4 file.txt

# Показать прогресс
xz -v file.txt
```

### Просмотр без распаковки

```bash
# Просмотреть
xzcat file.txt.xz
xz -dc file.txt.xz

# Поиск
xzgrep "pattern" file.txt.xz

# Сравнение
xzdiff file1.txt.xz file2.txt.xz
```

### Информация о файле

```bash
# Показать информацию
xz -l file.txt.xz
xz --list file.txt.xz

# Подробно
xz -lv file.txt.xz

# Проверить целостность
xz -t file.txt.xz
```

## Сравнение утилит

### Тест на текстовом файле

```bash
# Оригинал: 10 MB текстовый файл
ls -lh file.txt
# 10M file.txt

gzip -k file.txt
# 2.5M file.txt.gz (время: 0.5s)

bzip2 -k file.txt
# 1.8M file.txt.bz2 (время: 2s)

xz -k file.txt
# 1.2M file.txt.xz (время: 5s)
```

### Когда использовать

| Утилита | Когда использовать |
|---------|-------------------|
| gzip | Повседневное использование, быстрый доступ |
| bzip2 | Архивирование, когда важен размер |
| xz | Дистрибутивы, пакеты, долгосрочное хранение |

## Современные альтернативы

### zstd (Zstandard)

```bash
# Установка
sudo apt install zstd

# Сжатие
zstd file.txt           # file.txt.zst
zstd -19 file.txt       # максимальное сжатие

# Распаковка
zstd -d file.txt.zst
unzstd file.txt.zst

# Преимущества:
# - Быстрее gzip при лучшем сжатии
# - Хорошо масштабируется
# - Поддержка потоков
```

### lz4

```bash
# Установка
sudo apt install lz4

# Сжатие (очень быстро)
lz4 file.txt            # file.txt.lz4

# Распаковка
lz4 -d file.txt.lz4

# Идеально для:
# - Логов в реальном времени
# - Кеширования
# - Когда важна скорость
```

### pigz - параллельный gzip

```bash
# Установка
sudo apt install pigz

# Использует все ядра CPU
pigz file.txt
pigz -p 4 file.txt      # 4 потока

# Совместим с gzip
gunzip file.txt.gz      # работает
```

## Практические примеры

### Сжатие логов

```bash
# Сжать старые логи
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;

# Ротация с сжатием (в logrotate)
# compress
# compresscmd /bin/gzip
```

### Сжатие бекапов

```bash
# Для большого файла
pg_dump database | gzip > backup.sql.gz

# Для максимального сжатия
pg_dump database | xz -9 > backup.sql.xz

# С параллельной компрессией
pg_dump database | pigz > backup.sql.gz
```

### Конвертация форматов

```bash
# bz2 -> gz
bzip2 -dc file.txt.bz2 | gzip > file.txt.gz

# gz -> xz
gzip -dc file.txt.gz | xz > file.txt.xz
```

### Работа в пайплайнах

```bash
# Сжатие на лету
tar cf - directory/ | gzip > archive.tar.gz

# Передача по сети со сжатием
tar cf - directory/ | gzip | ssh user@server "cat > archive.tar.gz"

# Распаковка на лету
curl https://example.com/file.gz | gzip -d | process_data
```

## Резюме

| Команда | Сжатие | Распаковка |
|---------|--------|------------|
| gzip | `gzip file` | `gunzip file.gz` |
| bzip2 | `bzip2 file` | `bunzip2 file.bz2` |
| xz | `xz file` | `unxz file.xz` |
| zstd | `zstd file` | `unzstd file.zst` |
| lz4 | `lz4 file` | `lz4 -d file.lz4` |

### Общие опции

| Опция | Описание |
|-------|----------|
| `-d` | Распаковать |
| `-k` | Сохранить оригинал |
| `-c` | Вывод в stdout |
| `-f` | Принудительно |
| `-t` | Проверить целостность |
| `-v` | Подробный вывод |
| `-1...-9` | Уровень сжатия |

---

[prev: 04-xargs](../04-searching-files/04-xargs.md) | [next: 02-tar](./02-tar.md)
