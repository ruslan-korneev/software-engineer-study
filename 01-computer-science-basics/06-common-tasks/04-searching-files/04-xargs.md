# xargs - построение и выполнение команд из стандартного ввода

## Что такое xargs?

**xargs** (extended arguments) - утилита, которая читает данные из stdin и преобразует их в аргументы для указанной команды. Это позволяет эффективно обрабатывать большие списки файлов.

## Зачем нужен xargs?

### Проблема: ограничение длины командной строки

```bash
# Это может не работать при большом количестве файлов
rm $(find . -name "*.tmp")
# bash: /bin/rm: Argument list too long

# xargs решает эту проблему, разбивая на части
find . -name "*.tmp" | xargs rm
```

### Проблема: пустой ввод

```bash
# Без файлов rm выполнится без аргументов
find . -name "*.nothing" | xargs rm
# rm: missing operand

# С -r (--no-run-if-empty) команда не выполнится
find . -name "*.nothing" | xargs -r rm
```

## Базовое использование

```bash
# Простейший пример
echo "file1 file2 file3" | xargs ls -l

# Из find
find . -name "*.txt" | xargs wc -l

# Из файла со списком
cat files.txt | xargs rm

# Из любого вывода
grep -l "error" *.log | xargs head -5
```

## Основные опции

### -n - количество аргументов за раз

```bash
# Передавать по 1 аргументу
echo "a b c d e" | xargs -n 1 echo
# a
# b
# c
# d
# e

# По 2 аргумента
echo "a b c d e f" | xargs -n 2 echo
# a b
# c d
# e f

# Практический пример: обработка парами
echo "old1 new1 old2 new2" | xargs -n 2 mv
```

### -I - замена placeholder

```bash
# Использовать {} как placeholder
echo "file.txt" | xargs -I {} cp {} /backup/{}

# Любой placeholder
echo "file.txt" | xargs -I FILE mv FILE FILE.bak

# Несколько использований
ls *.txt | xargs -I {} sh -c 'echo "Processing {}"; wc -l {}'

# С find
find . -name "*.txt" | xargs -I {} cp {} /backup/
```

### -0 / --null - нулевой разделитель

```bash
# Проблема: файлы с пробелами
ls
# "my file.txt"  "another file.txt"

find . -name "*.txt" | xargs rm   # ОШИБКА: попытается удалить "my", "file.txt"

# Решение: использовать \0 как разделитель
find . -name "*.txt" -print0 | xargs -0 rm

# Работает с любыми именами файлов, включая:
# - пробелы
# - переносы строк
# - кавычки
# - специальные символы
```

### -P - параллельное выполнение

```bash
# Запустить 4 процесса параллельно
find . -name "*.jpg" -print0 | xargs -0 -P 4 -n 1 convert_image

# Полезно для CPU-интенсивных задач
cat urls.txt | xargs -P 10 -n 1 curl -O

# Комбинация с -n
find . -name "*.gz" -print0 | xargs -0 -P 4 -n 5 gzip -d
```

### -t / --verbose - показать команды

```bash
# Показать какая команда выполняется
echo "file1.txt file2.txt" | xargs -t rm
# rm file1.txt file2.txt

# Полезно для отладки
find . -name "*.tmp" -print0 | xargs -0 -t rm
```

### -p / --interactive - интерактивный режим

```bash
# Спрашивать подтверждение
find . -name "*.tmp" | xargs -p rm
# rm file1.tmp file2.tmp ?...y

# Комбинация с -n 1 для каждого файла отдельно
find . -name "*.tmp" | xargs -p -n 1 rm
```

### -r / --no-run-if-empty

```bash
# Не выполнять команду если ввод пустой (GNU xargs)
find . -name "*.nothing" | xargs -r rm

# В BSD/macOS -r не нужен (это поведение по умолчанию)
```

### -L - по строкам

```bash
# Одна строка = один аргумент
cat urls.txt | xargs -L 1 curl -O

# Несколько строк за раз
cat urls.txt | xargs -L 5 download_batch
```

### -d - разделитель

```bash
# Использовать определённый разделитель
echo "a:b:c:d" | xargs -d ':' echo
# a b c d

# Разделитель - перенос строки
cat files.txt | xargs -d '\n' ls -l
```

## Практические примеры

### Удаление файлов

```bash
# Удалить все .tmp файлы (безопасно)
find . -name "*.tmp" -print0 | xargs -0 rm -f

# С предварительным просмотром
find . -name "*.tmp" -print0 | xargs -0 ls -l

# С подтверждением
find . -name "*.tmp" | xargs -p rm
```

### Копирование файлов

```bash
# Скопировать найденные файлы
find . -name "*.conf" -print0 | xargs -0 -I {} cp {} /backup/

# С сохранением структуры
find . -name "*.py" -print0 | xargs -0 cp --parents -t /backup/
```

### Изменение прав

```bash
# Установить права на все скрипты
find . -name "*.sh" -print0 | xargs -0 chmod +x

# Исправить права на файлы
find /var/www -type f -print0 | xargs -0 chmod 644

# Исправить права на директории
find /var/www -type d -print0 | xargs -0 chmod 755
```

### Поиск в файлах

```bash
# grep во всех .py файлах
find . -name "*.py" -print0 | xargs -0 grep "import"

# Подсчёт строк кода
find . -name "*.py" -print0 | xargs -0 wc -l | tail -1

# Поиск TODO в проекте
find . -name "*.js" -print0 | xargs -0 grep -n "TODO"
```

### Архивирование

```bash
# Создать архив из списка файлов
find . -name "*.log" -mtime +7 -print0 | xargs -0 tar -czvf old_logs.tar.gz

# Добавить файлы в существующий архив
find . -name "*.txt" -print0 | xargs -0 tar -rvf archive.tar
```

### Параллельная обработка

```bash
# Конвертация изображений (4 процесса)
find . -name "*.png" -print0 | xargs -0 -P 4 -n 1 convert_to_jpg

# Массовая загрузка файлов
cat urls.txt | xargs -P 10 -n 1 wget -q

# Сжатие файлов параллельно
find . -name "*.log" -print0 | xargs -0 -P 8 gzip
```

### Работа с git

```bash
# Добавить все изменённые .py файлы
git status --porcelain | grep '\.py$' | cut -c4- | xargs git add

# Удалить untracked файлы (осторожно!)
git ls-files --others --exclude-standard | xargs -r rm

# Искать в истории
git log --oneline --all | head -100 | xargs -I {} sh -c 'echo "=== {} ===" && git show {} --stat'
```

### Обработка CSV/данных

```bash
# Извлечь колонку и обработать
cat data.csv | cut -d',' -f2 | xargs -n 1 process_item

# Преобразовать список в одну строку
cat items.txt | xargs echo

# Добавить разделитель
cat items.txt | xargs printf '%s,'
```

## Сложные примеры

### Выполнение shell-команд

```bash
# Использовать sh -c для сложных команд
find . -name "*.txt" | xargs -I {} sh -c 'wc -l {} && head -1 {}'

# Переменные внутри
ls *.log | xargs -I {} sh -c 'base=$(basename {} .log); mv {} ${base}.txt'
```

### Безопасная замена

```bash
# Кавычки вокруг {}
find . -name "*.txt" | xargs -I {} sh -c 'process "{}"'

# Ещё безопаснее - передать как аргумент
find . -name "*.txt" -print0 | xargs -0 -I {} sh -c 'process "$1"' _ {}
```

### Работа с множественными командами

```bash
# Несколько команд для каждого файла
find . -name "*.py" | xargs -I {} sh -c '
    echo "Processing {}..."
    python -m py_compile {}
    echo "Done with {}"
'
```

## Альтернативы xargs

### GNU Parallel

```bash
# Установка
sudo apt install parallel

# Параллельная обработка (проще чем xargs -P)
find . -name "*.jpg" | parallel convert {} {.}.png

# С прогрессом
find . -name "*.gz" | parallel --progress gzip -d {}

# На нескольких машинах
parallel --sshloginfile servers.txt process {} ::: file1 file2 file3
```

### find -exec +

```bash
# Эквивалент xargs (для find)
find . -name "*.txt" -exec wc -l {} +

# Но xargs более гибкий
find . -name "*.txt" -print0 | xargs -0 -P 4 wc -l
```

## Частые ошибки

### 1. Имена с пробелами

```bash
# НЕПРАВИЛЬНО
find . -name "*.txt" | xargs rm    # сломается на "my file.txt"

# ПРАВИЛЬНО
find . -name "*.txt" -print0 | xargs -0 rm
```

### 2. Пустой ввод

```bash
# НЕПРАВИЛЬНО (в некоторых системах)
find . -name "*.nothing" | xargs rm
# rm: missing operand

# ПРАВИЛЬНО
find . -name "*.nothing" | xargs -r rm
```

### 3. Лимит аргументов

```bash
# МОЖЕТ НЕ РАБОТАТЬ для очень большого количества
ls *.log | xargs cat > all.log

# НАДЁЖНЕЕ
find . -name "*.log" -print0 | xargs -0 cat > all.log
```

## Резюме

| Опция | Описание |
|-------|----------|
| `-n N` | По N аргументов за раз |
| `-I {}` | Placeholder для подстановки |
| `-0` | Использовать \0 как разделитель |
| `-P N` | N параллельных процессов |
| `-t` | Показать выполняемые команды |
| `-p` | Спрашивать подтверждение |
| `-r` | Не выполнять если ввод пустой |
| `-L N` | По N строк за раз |
| `-d D` | Использовать D как разделитель |

### Шаблоны использования

```bash
# Безопасная обработка файлов
find ... -print0 | xargs -0 command

# С подстановкой
find ... | xargs -I {} command {} args

# Параллельно
find ... -print0 | xargs -0 -P 4 -n 1 command

# Сложная команда
find ... | xargs -I {} sh -c 'complex command with {}'
```
