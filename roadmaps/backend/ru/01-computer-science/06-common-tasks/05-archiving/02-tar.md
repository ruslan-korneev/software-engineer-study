# tar - архивирование файлов

## Что такое tar?

**tar** (tape archive) - утилита для объединения множества файлов в один архив. Изначально создана для записи на магнитные ленты, сейчас является стандартом для архивирования в Unix/Linux.

**Важно:** tar сам по себе не сжимает данные, а только объединяет файлы. Для сжатия используется в комбинации с gzip, bzip2, xz.

## Основные операции

| Опция | Описание |
|-------|----------|
| `-c` | Create - создать архив |
| `-x` | Extract - извлечь файлы |
| `-t` | List - показать содержимое |
| `-r` | Append - добавить файлы |
| `-u` | Update - обновить файлы |

## Базовое использование

### Создание архива

```bash
# Создать архив
tar -cvf archive.tar directory/
tar -cvf archive.tar file1.txt file2.txt

# Опции:
# -c = create (создать)
# -v = verbose (подробный вывод)
# -f = file (имя архива)

# Без verbose (тихо)
tar -cf archive.tar directory/
```

### Просмотр содержимого

```bash
# Показать список файлов
tar -tf archive.tar

# С подробной информацией
tar -tvf archive.tar

# Пример вывода:
# drwxr-xr-x user/group    0 2024-01-15 10:30 directory/
# -rw-r--r-- user/group 1234 2024-01-15 10:30 directory/file.txt
```

### Извлечение файлов

```bash
# Извлечь все файлы в текущую директорию
tar -xvf archive.tar

# Извлечь в указанную директорию
tar -xvf archive.tar -C /destination/path/

# Извлечь конкретные файлы
tar -xvf archive.tar path/to/file.txt
tar -xvf archive.tar directory/

# Извлечь по шаблону
tar -xvf archive.tar --wildcards "*.txt"
```

## Сжатие архивов

### gzip (.tar.gz или .tgz)

```bash
# Создать сжатый архив
tar -czvf archive.tar.gz directory/

# Извлечь
tar -xzvf archive.tar.gz

# Просмотреть
tar -tzvf archive.tar.gz

# Опция -z = gzip
```

### bzip2 (.tar.bz2)

```bash
# Создать
tar -cjvf archive.tar.bz2 directory/

# Извлечь
tar -xjvf archive.tar.bz2

# Опция -j = bzip2
```

### xz (.tar.xz)

```bash
# Создать
tar -cJvf archive.tar.xz directory/

# Извлечь
tar -xJvf archive.tar.xz

# Опция -J = xz
```

### Автоматическое определение

```bash
# При извлечении tar автоматически определяет формат
tar -xf archive.tar.gz
tar -xf archive.tar.bz2
tar -xf archive.tar.xz

# Опция -a автоматически выбирает сжатие при создании
tar -cavf archive.tar.gz directory/    # gzip
tar -cavf archive.tar.xz directory/    # xz
```

## Дополнительные опции

### Исключение файлов

```bash
# Исключить файлы по шаблону
tar -cvf archive.tar --exclude="*.log" directory/
tar -cvf archive.tar --exclude="node_modules" directory/

# Несколько исключений
tar -cvf archive.tar \
    --exclude="*.log" \
    --exclude="*.tmp" \
    --exclude=".git" \
    directory/

# Исключить из файла
tar -cvf archive.tar --exclude-from=exclude.txt directory/
```

### Включение только определённых файлов

```bash
# Архивировать только .txt файлы
find . -name "*.txt" | tar -cvf archive.tar -T -

# Из списка файлов
tar -cvf archive.tar -T files.txt
tar -cvf archive.tar --files-from=files.txt
```

### Сохранение атрибутов

```bash
# Сохранить права, владельца, время
tar -cvpf archive.tar directory/
# -p = preserve permissions

# При извлечении сохранить владельца (нужен root)
sudo tar -xvpf archive.tar

# Сохранить все атрибуты (ACL, SELinux, xattr)
tar --xattrs --selinux --acls -cvf archive.tar directory/
```

### Работа с большими архивами

```bash
# Разбить на части
tar -cvf - directory/ | split -b 100M - archive.tar.part

# Объединить и извлечь
cat archive.tar.part* | tar -xvf -

# Или с gzip
tar -czvf - directory/ | split -b 100M - archive.tar.gz.part
cat archive.tar.gz.part* | tar -xzvf -
```

### Прогресс выполнения

```bash
# С pv (pipe viewer)
tar -cf - directory/ | pv | gzip > archive.tar.gz

# Показать размер
tar -cf - directory/ | pv -s $(du -sb directory/ | cut -f1) | gzip > archive.tar.gz

# Встроенный checkpoint
tar --checkpoint=1000 --checkpoint-action=dot -cvf archive.tar directory/
```

## Обновление архивов

### Добавление файлов

```bash
# Добавить файл в существующий архив (без сжатия!)
tar -rvf archive.tar newfile.txt

# Обновить изменившиеся файлы
tar -uvf archive.tar directory/

# ВАЖНО: -r и -u не работают со сжатыми архивами!
# Для .tar.gz нужно распаковать, изменить, сжать заново
```

### Удаление файлов из архива

```bash
# Удалить файл из архива (только несжатый .tar)
tar --delete -f archive.tar path/to/file.txt

# Для сжатых - пересоздать архив
tar -tzf archive.tar.gz | grep -v "unwanted_file" | tar -T - -czvf new_archive.tar.gz
```

## Инкрементальное резервное копирование

```bash
# Создать полный бекап с файлом снимка
tar --listed-incremental=snapshot.snar -cvzf backup-full.tar.gz directory/

# Инкрементальный бекап (только изменения)
tar --listed-incremental=snapshot.snar -cvzf backup-incr-1.tar.gz directory/

# Ещё один инкрементальный
tar --listed-incremental=snapshot.snar -cvzf backup-incr-2.tar.gz directory/

# Восстановление (в порядке создания)
tar -xzvf backup-full.tar.gz
tar -xzvf backup-incr-1.tar.gz
tar -xzvf backup-incr-2.tar.gz
```

## Сетевые операции

### Копирование через SSH

```bash
# Скопировать директорию на удалённый сервер
tar -czvf - directory/ | ssh user@server "cat > archive.tar.gz"

# Распаковать на удалённом сервере
tar -czvf - directory/ | ssh user@server "tar -xzvf - -C /destination/"

# Скопировать с удалённого сервера
ssh user@server "tar -czvf - /path/to/dir/" | tar -xzvf -

# Синхронизация между серверами
ssh user@source "tar -cf - /path/" | ssh user@dest "tar -xf - -C /path/"
```

### Копирование на удалённый хост

```bash
# Через netcat (быстрее SSH, но небезопасно)
# На приёмнике:
nc -l -p 9999 | tar -xzvf -

# На отправителе:
tar -czvf - directory/ | nc receiver_host 9999
```

## Практические примеры

### Бекап домашней директории

```bash
# Полный бекап с исключениями
tar -czvf ~/backup-$(date +%Y%m%d).tar.gz \
    --exclude=".cache" \
    --exclude="node_modules" \
    --exclude=".local/share/Trash" \
    ~/
```

### Бекап веб-сервера

```bash
# Сайт + база данных
tar -czvf backup.tar.gz /var/www/html/
mysqldump -u root -p database | gzip > database.sql.gz

# Всё вместе
tar -czvf full_backup.tar.gz /var/www/html/ database.sql.gz
```

### Миграция системы

```bash
# Создать архив системы (от root)
sudo tar -czvf system_backup.tar.gz \
    --exclude=/proc \
    --exclude=/sys \
    --exclude=/dev \
    --exclude=/run \
    --exclude=/tmp \
    --exclude=/mnt \
    --exclude=/media \
    --exclude="*.tar.gz" \
    /
```

### Архивирование с проверкой

```bash
# Создать и сразу проверить
tar -czvf archive.tar.gz directory/ && tar -tzvf archive.tar.gz > /dev/null

# С контрольной суммой
tar -czvf archive.tar.gz directory/
md5sum archive.tar.gz > archive.tar.gz.md5

# Проверить
md5sum -c archive.tar.gz.md5
```

### Извлечение определённых типов файлов

```bash
# Только .py файлы
tar -xzvf archive.tar.gz --wildcards "*.py"

# Только определённая директория
tar -xzvf archive.tar.gz "project/src/"

# Список файлов для извлечения из файла
tar -xzvf archive.tar.gz -T extract_list.txt
```

## Сравнение синтаксиса

### Традиционный синтаксис

```bash
tar cvf archive.tar directory/    # без дефиса
tar -cvf archive.tar directory/   # с дефисом
tar --create --verbose --file=archive.tar directory/  # длинные опции
```

### Опции файла

```bash
# -f всегда должен быть последним с коротким синтаксисом
tar -cvf archive.tar dir/    # правильно
tar -cfv archive.tar dir/    # НЕПРАВИЛЬНО

# Или используйте --file=
tar -cv --file=archive.tar dir/
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Создать .tar | `tar -cvf archive.tar dir/` |
| Создать .tar.gz | `tar -czvf archive.tar.gz dir/` |
| Создать .tar.bz2 | `tar -cjvf archive.tar.bz2 dir/` |
| Создать .tar.xz | `tar -cJvf archive.tar.xz dir/` |
| Извлечь | `tar -xvf archive.tar` |
| Извлечь в папку | `tar -xvf archive.tar -C /path/` |
| Просмотреть содержимое | `tar -tvf archive.tar` |
| Добавить файл | `tar -rvf archive.tar file` |
| С исключениями | `tar --exclude="*.log" -czvf ...` |
| Инкрементальный | `tar --listed-incremental=snap.snar ...` |

### Расширения файлов

| Расширение | Формат |
|------------|--------|
| `.tar` | Без сжатия |
| `.tar.gz`, `.tgz` | gzip |
| `.tar.bz2`, `.tbz2` | bzip2 |
| `.tar.xz`, `.txz` | xz |
| `.tar.zst` | zstd |
