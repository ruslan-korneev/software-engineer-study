# Перенаправление вывода

## Операторы перенаправления stdout

| Оператор | Действие |
|----------|----------|
| `>` | Перенаправить stdout в файл (перезаписать) |
| `>>` | Перенаправить stdout в файл (добавить) |
| `1>` | То же что `>` (явное указание FD) |
| `1>>` | То же что `>>` |

## Перезапись файла (>)

Оператор `>` перенаправляет stdout в файл, **перезаписывая** его содержимое:

```bash
$ echo "Hello" > file.txt
$ cat file.txt
Hello

$ echo "World" > file.txt
$ cat file.txt
World                      # "Hello" перезаписано
```

### Примеры
```bash
# Сохранить листинг
$ ls -la > listing.txt

# Сохранить дату
$ date > timestamp.txt

# Вывод команды в файл
$ ps aux > processes.txt
```

## Добавление в файл (>>)

Оператор `>>` добавляет данные в конец файла:

```bash
$ echo "First line" > log.txt
$ echo "Second line" >> log.txt
$ echo "Third line" >> log.txt
$ cat log.txt
First line
Second line
Third line
```

### Примеры
```bash
# Добавлять логи
$ echo "$(date): Script started" >> app.log
$ echo "$(date): Script finished" >> app.log

# Накопление данных
$ ls /etc >> all-files.txt
$ ls /var >> all-files.txt
```

## Создание и очистка файлов

### Создать пустой файл
```bash
$ > newfile.txt            # создаёт пустой файл
$ : > newfile.txt          # альтернатива
```

### Очистить существующий файл
```bash
$ > logfile.txt            # очищает файл, не удаляя
$ cat /dev/null > file.txt # альтернатива
$ truncate -s 0 file.txt   # ещё вариант
```

## Перенаправление в /dev/null

`/dev/null` — специальный файл, который отбрасывает все данные ("чёрная дыра"):

```bash
$ command > /dev/null      # отбросить stdout
$ ls nonexistent > /dev/null
ls: cannot access 'nonexistent': No such file or directory
# Ошибка (stderr) всё ещё видна

$ command &> /dev/null     # отбросить и stdout, и stderr
```

### Когда использовать
```bash
# Проверить успешность команды без вывода
$ grep -q pattern file.txt > /dev/null && echo "Found"

# Запустить фоновую задачу без вывода
$ background-task > /dev/null 2>&1 &
```

## Комбинирование с командами

### Сохранить и увидеть (tee)
```bash
$ ls | tee listing.txt     # вывод на экран И в файл
file1
file2
$ cat listing.txt
file1
file2
```

С добавлением:
```bash
$ echo "new data" | tee -a log.txt
```

### Цепочка команд с записью
```bash
$ cat file.txt | grep pattern | sort > result.txt
```

## Перенаправление от нескольких команд

### Группировка в скобках
```bash
$ (echo "Header"; ls; echo "Footer") > output.txt
```

### Фигурные скобки (в текущем shell)
```bash
$ { echo "Header"; ls; echo "Footer"; } > output.txt
```

## Here Document (<<)

Способ передать многострочный текст:

```bash
$ cat << EOF > file.txt
Line 1
Line 2
Line 3
EOF

$ cat file.txt
Line 1
Line 2
Line 3
```

### С переменными
```bash
$ NAME="World"
$ cat << EOF > greeting.txt
Hello, $NAME!
Today is $(date).
EOF
```

### Без интерпретации (<<'EOF')
```bash
$ cat << 'EOF' > script.sh
echo $HOME
echo $(date)
EOF
# Переменные НЕ раскрываются
```

## Here String (<<<)

Однострочный вариант:

```bash
$ cat <<< "Hello, World"
Hello, World

$ wc -w <<< "one two three"
3
```

## Перенаправление в другой дескриптор

### Stdout в stderr
```bash
$ echo "Error message" >&2
```

### Stderr в stdout
```bash
$ command 2>&1             # stderr туда же, куда stdout
```

## Практические сценарии

### Логирование скрипта
```bash
#!/bin/bash
LOG="script.log"
echo "=== $(date) ===" >> "$LOG"
echo "Starting script" >> "$LOG"
some_command >> "$LOG" 2>&1
echo "Script finished" >> "$LOG"
```

### Создание отчёта
```bash
{
    echo "System Report - $(date)"
    echo "===================="
    echo
    echo "Disk usage:"
    df -h
    echo
    echo "Memory:"
    free -h
} > report.txt
```

### Архивирование с логом
```bash
tar -czvf backup.tar.gz /data 2>&1 | tee backup.log
```

## Важные замечания

1. **`>` перезаписывает без предупреждения** — будьте осторожны!
   ```bash
   $ set -o noclobber    # запретить перезапись существующих файлов
   $ echo "test" > existing.txt
   bash: existing.txt: cannot overwrite existing file
   $ echo "test" >| existing.txt   # принудительная перезапись
   ```

2. **Порядок перенаправления важен**:
   ```bash
   $ command > file 2>&1     # stderr → stdout → file (правильно)
   $ command 2>&1 > file     # stdout → file, stderr → терминал
   ```

3. **Файл создаётся до выполнения команды**:
   ```bash
   $ cat file.txt > file.txt   # ОШИБКА! Файл очистится до чтения
   ```
