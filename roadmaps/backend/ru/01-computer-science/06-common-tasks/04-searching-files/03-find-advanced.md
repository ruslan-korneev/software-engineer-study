# find - расширенные возможности

[prev: 02-find-basics](./02-find-basics.md) | [next: 04-xargs](./04-xargs.md)

---
## Действия над найденными файлами

### -print (по умолчанию)

```bash
# Вывести имена файлов (действие по умолчанию)
find . -name "*.txt" -print

# С нулевым разделителем (для xargs -0)
find . -name "*.txt" -print0
```

### -exec - выполнить команду

```bash
# Синтаксис: -exec command {} \;
# {} заменяется на имя найденного файла
# \; - конец команды

# Показать информацию о файлах
find . -name "*.txt" -exec ls -l {} \;

# Удалить файлы
find . -name "*.tmp" -exec rm {} \;

# Скопировать файлы
find . -name "*.conf" -exec cp {} /backup/ \;

# Изменить права
find . -name "*.sh" -exec chmod +x {} \;

# grep в файлах
find . -name "*.py" -exec grep -l "import os" {} \;
```

### -exec с {} +

Более эффективный вариант - передаёт несколько файлов одной команде:

```bash
# {} + - передать все файлы одной командой (как xargs)
find . -name "*.txt" -exec ls -l {} +

# Сравнение:
# {} \; - ls -l file1; ls -l file2; ls -l file3
# {} +  - ls -l file1 file2 file3
```

### -execdir - выполнить в директории файла

```bash
# Выполняет команду в директории, где находится файл
find . -name "*.py" -execdir python {} \;

# Безопаснее чем -exec (защита от подстановки путей)
find . -name "install.sh" -execdir ./install.sh \;
```

### -ok - с подтверждением

```bash
# Спрашивает подтверждение перед каждым действием
find . -name "*.tmp" -ok rm {} \;
# < rm ... ./file.tmp > ? y
```

### -delete - удаление

```bash
# Встроенное удаление (эффективнее -exec rm)
find . -name "*.tmp" -delete

# ВАЖНО: -delete должен быть последним!
# И требует -depth (неявно включается)

# Удалить пустые директории
find . -type d -empty -delete
```

## Опции форматирования вывода

### -printf - форматированный вывод

```bash
# Кастомный формат вывода
find . -name "*.txt" -printf "%p %s\n"

# Директивы:
# %p - полный путь
# %f - имя файла
# %h - директория файла
# %s - размер в байтах
# %k - размер в КБ
# %m - права (восьмеричные)
# %M - права (как ls)
# %u - имя владельца
# %g - имя группы
# %t - время изменения
# %Tk - время в формате (k = формат strftime)
# %n - количество ссылок

# Примеры:
find . -type f -printf "%s %p\n" | sort -n    # сортировка по размеру
find . -type f -printf "%T+ %p\n" | sort      # сортировка по времени
find . -printf "%M %u %s %p\n"                # как ls -l
find . -type f -printf "%f\n"                  # только имена файлов
```

### Форматы времени для %T

```bash
# %Tk где k - спецификатор формата
find . -printf "%T+ %p\n"        # 2024-01-15+12:30:45.0000000000
find . -printf "%TY-%Tm-%Td %p\n" # 2024-01-15
find . -printf "%TH:%TM %p\n"     # 12:30
```

### -ls - вывод в формате ls -l

```bash
find . -name "*.conf" -ls

# Эквивалент:
find . -name "*.conf" -exec ls -dils {} \;
```

## Работа с символическими ссылками

```bash
# По умолчанию find не следует по ссылкам

# -L - следовать по символическим ссылкам
find -L /path -name "*.txt"

# -P - не следовать (по умолчанию)
find -P /path -name "*.txt"

# -H - следовать только если ссылка в аргументе командной строки
find -H /symlink/path -name "*.txt"

# Найти битые ссылки
find . -xtype l

# Найти ссылки, указывающие на определённый файл
find . -lname "target_pattern"
find . -samefile /path/to/file
```

## Оптимизация поиска

### -depth и -d

```bash
# Обрабатывать содержимое директории до самой директории
find . -depth -name "*.tmp" -delete

# Полезно при удалении (сначала файлы, потом директории)
find . -depth -type d -empty -delete
```

### -mount / -xdev

```bash
# Не переходить на другие файловые системы
find / -mount -name "*.log"
find / -xdev -name "*.log"    # синоним
```

### -maxdepth перед другими опциями

```bash
# Правильно - maxdepth вначале (быстрее)
find . -maxdepth 3 -name "*.txt"

# Не оптимально - сначала обойдёт все файлы
find . -name "*.txt" -maxdepth 3  # работает, но менее эффективно
```

## Комбинация с xargs

### Зачем xargs?

`xargs` преобразует ввод в аргументы команды. Эффективнее чем `-exec {} \;`:

```bash
# Вместо множества вызовов rm
find . -name "*.tmp" -exec rm {} \;

# Один вызов rm со всеми файлами
find . -name "*.tmp" | xargs rm

# Аналогично:
find . -name "*.tmp" -exec rm {} +
```

### Безопасная работа с именами файлов

```bash
# Проблема: имена с пробелами и спецсимволами
find . -name "*.txt" | xargs ls -l    # сломается на "file name.txt"

# Решение: нулевой разделитель
find . -name "*.txt" -print0 | xargs -0 ls -l
```

### Ограничение количества аргументов

```bash
# Передавать по 10 файлов за раз
find . -name "*.txt" -print0 | xargs -0 -n 10 command

# Параллельное выполнение
find . -name "*.txt" -print0 | xargs -0 -P 4 -n 10 command
# -P 4 - 4 параллельных процесса
```

### Подстановка в середину команды

```bash
# Стандартно аргументы добавляются в конец
find . -name "*.txt" | xargs wc -l

# Подставить в определённое место
find . -name "*.txt" | xargs -I {} cp {} /backup/
find . -name "*.txt" | xargs -I FILE mv FILE FILE.bak
```

## Практические сценарии

### Массовое переименование

```bash
# Изменить расширение .txt на .md
find . -name "*.txt" -exec sh -c 'mv "$1" "${1%.txt}.md"' _ {} \;

# С xargs и rename
find . -name "*.txt" -print0 | xargs -0 rename 's/\.txt$/.md/'

# Добавить префикс
find . -name "*.jpg" -exec sh -c 'mv "$1" "$(dirname "$1")/prefix_$(basename "$1")"' _ {} \;
```

### Резервное копирование с сохранением структуры

```bash
# Скопировать только .conf файлы с сохранением путей
find /etc -name "*.conf" -exec cp --parents {} /backup/ \;

# С использованием rsync
find . -name "*.py" -print0 | rsync -av --files-from=- --from0 . /backup/
```

### Поиск и архивирование

```bash
# Архивировать все .log файлы старше 7 дней
find /var/log -name "*.log" -mtime +7 -print0 | \
    tar --null -T - -czvf old_logs.tar.gz

# Удалить после архивирования
find /var/log -name "*.log" -mtime +7 -delete
```

### Очистка проекта

```bash
# Удалить все временные файлы
find . \( -name "*.pyc" -o -name "*.pyo" -o -name "__pycache__" \) -delete

# Node.js проект
find . -name "node_modules" -type d -prune -exec rm -rf {} +

# Удалить пустые директории
find . -type d -empty -delete
```

### Синхронизация изменений

```bash
# Найти файлы изменённые за последний час и скопировать
find . -mmin -60 -type f -exec cp --parents {} /backup/ \;

# С rsync
find . -mmin -60 -type f -print0 | \
    rsync -av --files-from=- --from0 . /backup/
```

### Проверка прав доступа

```bash
# Найти файлы с неправильными правами
find /var/www -type f ! -perm 644 -ls
find /var/www -type d ! -perm 755 -ls

# Исправить права
find /var/www -type f ! -perm 644 -exec chmod 644 {} +
find /var/www -type d ! -perm 755 -exec chmod 755 {} +
```

### Статистика проекта

```bash
# Подсчёт строк кода
find . -name "*.py" -type f -exec wc -l {} + | tail -1

# Количество файлов по типам
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Размер по типам файлов
find . -type f -printf "%s %f\n" | awk -F. '{sum[$NF]+=$1} END {for(i in sum) print sum[i], i}' | sort -rn
```

### Поиск дубликатов (по размеру)

```bash
# Найти файлы одинакового размера
find . -type f -printf "%s %p\n" | sort -n | uniq -D -w 10

# С проверкой по хешу
find . -type f -exec md5sum {} + | sort | uniq -d -w 32
```

## Отладка find

```bash
# Показать что find пытается сделать
find . -name "*.txt" -exec echo "Found: {}" \;

# Сухой прогон удаления
find . -name "*.tmp" -exec echo rm {} \;

# Проверить какие файлы будут обработаны
find . -name "*.txt" | head -20

# Использовать -ok для интерактивного подтверждения
find . -name "*.tmp" -ok rm {} \;
```

## Резюме

| Действие | Команда |
|----------|---------|
| Выполнить команду | `-exec cmd {} \;` |
| Выполнить эффективно | `-exec cmd {} +` |
| Выполнить в директории | `-execdir cmd {} \;` |
| С подтверждением | `-ok cmd {} \;` |
| Удалить | `-delete` |
| Форматированный вывод | `-printf "format"` |
| Вывод как ls | `-ls` |
| С xargs | `find ... -print0 \| xargs -0 cmd` |

---

[prev: 02-find-basics](./02-find-basics.md) | [next: 04-xargs](./04-xargs.md)
