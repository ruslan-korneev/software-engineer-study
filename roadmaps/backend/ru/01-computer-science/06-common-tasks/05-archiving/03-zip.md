# zip и unzip - работа с ZIP-архивами

[prev: 02-tar](./02-tar.md) | [next: 04-rsync](./04-rsync.md)

---
## Что такое ZIP?

**ZIP** - популярный формат архивов, широко используемый в Windows и поддерживаемый во всех операционных системах. В отличие от tar + gzip, ZIP сжимает каждый файл отдельно, что позволяет извлекать отдельные файлы без распаковки всего архива.

## Установка

```bash
# Debian/Ubuntu
sudo apt install zip unzip

# RHEL/CentOS
sudo yum install zip unzip

# Fedora
sudo dnf install zip unzip

# macOS - предустановлено
```

## zip - создание архивов

### Базовое использование

```bash
# Создать архив из файлов
zip archive.zip file1.txt file2.txt

# Создать архив из директории
zip -r archive.zip directory/

# Опции:
# -r = рекурсивно (для директорий обязательно!)
```

### Уровни сжатия

```bash
# Без сжатия (только архивирование)
zip -0 archive.zip files/

# Максимальное сжатие
zip -9 archive.zip files/

# По умолчанию -6
zip archive.zip files/
```

### Обновление и добавление

```bash
# Добавить файл в существующий архив
zip archive.zip newfile.txt

# Обновить изменившиеся файлы
zip -u archive.zip directory/

# Freshen - обновить только существующие в архиве
zip -f archive.zip directory/

# Удалить файл из архива
zip -d archive.zip path/to/file.txt
```

### Исключение файлов

```bash
# Исключить по шаблону
zip -r archive.zip directory/ -x "*.log"
zip -r archive.zip directory/ -x "*.log" -x "*.tmp"

# Исключить директорию
zip -r archive.zip project/ -x "project/node_modules/*"
zip -r archive.zip project/ -x "*/.git/*"

# Из файла со списком исключений
zip -r archive.zip directory/ -x@exclude.txt
```

### Включение файлов по шаблону

```bash
# Только определённые файлы
zip archive.zip directory/*.txt
zip -r archive.zip directory/ -i "*.py"

# Из списка файлов
zip archive.zip -@ < files.txt
cat files.txt | zip archive.zip -@
```

### Защита паролем

```bash
# Установить пароль (запросит интерактивно)
zip -e archive.zip files/

# Пароль в командной строке (небезопасно)
zip -P "password" archive.zip files/

# Лучше использовать 7z для шифрования
```

### Разбиение на части

```bash
# Разбить на части по 100 МБ
zip -r -s 100m archive.zip directory/

# Результат: archive.zip, archive.z01, archive.z02...
```

### Дополнительные опции

```bash
# Показать что добавляется
zip -v archive.zip files/

# Тихий режим
zip -q archive.zip files/

# Переместить файлы в архив (удалить оригиналы)
zip -m archive.zip files/

# Сохранить символические ссылки
zip -y archive.zip symlinks/

# Сохранить структуру директорий
zip -r archive.zip dir/

# Не сохранять пути (плоская структура)
zip -j archive.zip path/to/files/*

# Тест после создания
zip -T archive.zip
```

## unzip - извлечение архивов

### Базовое использование

```bash
# Извлечь все файлы
unzip archive.zip

# Извлечь в указанную директорию
unzip archive.zip -d /destination/

# Показать содержимое без извлечения
unzip -l archive.zip

# Подробная информация
unzip -v archive.zip
```

### Выборочное извлечение

```bash
# Извлечь конкретный файл
unzip archive.zip path/to/file.txt

# Извлечь по шаблону
unzip archive.zip "*.txt"
unzip archive.zip "directory/*"

# Исключить файлы
unzip archive.zip -x "*.log"
unzip archive.zip -x "unwanted/*"
```

### Обработка конфликтов

```bash
# Перезаписать все без вопросов
unzip -o archive.zip

# Никогда не перезаписывать
unzip -n archive.zip

# Переименовывать новые файлы
unzip -B archive.zip

# Интерактивный режим (по умолчанию)
unzip archive.zip
# replace file.txt? [y]es, [n]o, [A]ll, [N]one, [r]ename:
```

### Проверка архива

```bash
# Тест целостности
unzip -t archive.zip

# Показать первые строки текстовых файлов
unzip -p archive.zip file.txt | head

# Извлечь в stdout
unzip -p archive.zip file.txt > output.txt
```

### Кодировка имён файлов

```bash
# Проблема с кириллицей из Windows
unzip -O CP866 archive.zip
unzip -O CP1251 archive.zip

# Показать имена файлов
unzip -l archive.zip
```

### Опции unzip

| Опция | Описание |
|-------|----------|
| `-l` | Показать содержимое |
| `-v` | Подробная информация |
| `-t` | Проверить целостность |
| `-d dir` | Директория назначения |
| `-o` | Перезаписывать без вопросов |
| `-n` | Не перезаписывать |
| `-j` | Игнорировать пути |
| `-p` | Вывод в stdout |
| `-q` | Тихий режим |
| `-x` | Исключить файлы |

## zipinfo - информация об архиве

```bash
# Краткая информация
zipinfo archive.zip

# Короткий формат (имена файлов)
zipinfo -1 archive.zip

# Средний формат
zipinfo -m archive.zip

# Длинный формат (как ls -l)
zipinfo -l archive.zip

# Статистика
zipinfo -h archive.zip
zipinfo -t archive.zip
```

## Сравнение ZIP и tar.gz

| Характеристика | ZIP | tar.gz |
|----------------|-----|--------|
| Сжатие | Каждый файл отдельно | Весь архив |
| Извлечение одного файла | Быстро | Нужно читать весь архив |
| Степень сжатия | Хуже | Лучше |
| Совместимость | Windows, Mac, Linux | Unix/Linux |
| Атрибуты Unix | Ограниченно | Полностью |
| Добавление файлов | Легко | Только в .tar |

## Практические примеры

### Архивирование проекта

```bash
# Исключить ненужное
zip -r project.zip project/ \
    -x "project/.git/*" \
    -x "project/node_modules/*" \
    -x "project/__pycache__/*" \
    -x "project/*.pyc"
```

### Резервное копирование

```bash
# Бекап с датой
zip -r backup_$(date +%Y%m%d).zip important_files/

# Инкрементальный бекап
zip -r backup.zip directory/ -DF
# -D = не добавлять записи для директорий
# -F = fix archive (freshen)
```

### Работа с несколькими архивами

```bash
# Объединить архивы
zip -r combined.zip first.zip second.zip

# Лучше - извлечь и заархивировать
unzip -d temp first.zip
unzip -d temp second.zip
zip -r combined.zip temp/
```

### Извлечение из множества архивов

```bash
# Распаковать все .zip в текущей директории
for f in *.zip; do unzip -d "${f%.zip}" "$f"; done

# Или проще
unzip "*.zip" -d output/
```

### Конвертация tar.gz в ZIP

```bash
# Распаковать tar.gz
tar -xzvf archive.tar.gz

# Создать ZIP
zip -r archive.zip extracted_directory/

# Одной командой через pipe
tar -xzf archive.tar.gz -O | zip archive.zip -
```

## Альтернативы: 7-Zip

### Установка p7zip

```bash
# Debian/Ubuntu
sudo apt install p7zip-full

# RHEL/CentOS
sudo yum install p7zip p7zip-plugins

# Fedora
sudo dnf install p7zip p7zip-plugins
```

### Использование 7z

```bash
# Создать архив
7z a archive.7z directory/

# Извлечь
7z x archive.7z

# Показать содержимое
7z l archive.7z

# Максимальное сжатие
7z a -mx=9 archive.7z directory/

# С паролем (шифрование AES-256)
7z a -p -mhe=on archive.7z directory/

# Поддержка разных форматов
7z a archive.zip directory/
7z x archive.tar.gz
```

### Преимущества 7z

- Лучшее сжатие (LZMA/LZMA2)
- Сильное шифрование (AES-256)
- Работает со многими форматами
- Разбиение на тома
- Самораспаковывающиеся архивы

## Резюме команд

| Задача | Команда |
|--------|---------|
| Создать ZIP | `zip archive.zip files` |
| Создать ZIP из директории | `zip -r archive.zip dir/` |
| Максимальное сжатие | `zip -9 archive.zip files` |
| Добавить файл | `zip archive.zip newfile` |
| Удалить файл | `zip -d archive.zip file` |
| С паролем | `zip -e archive.zip files` |
| Распаковать | `unzip archive.zip` |
| Распаковать в папку | `unzip archive.zip -d dest/` |
| Показать содержимое | `unzip -l archive.zip` |
| Проверить | `unzip -t archive.zip` |
| Перезаписать всё | `unzip -o archive.zip` |
| Не перезаписывать | `unzip -n archive.zip` |

---

[prev: 02-tar](./02-tar.md) | [next: 04-rsync](./04-rsync.md)
