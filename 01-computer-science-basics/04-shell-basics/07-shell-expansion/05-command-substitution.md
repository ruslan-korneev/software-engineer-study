# Command Substitution (подстановка команд)

## Что такое command substitution?

**Command substitution** — механизм, позволяющий использовать вывод команды как часть другой команды или присвоить его переменной.

```bash
$ echo "Today is $(date)"
Today is Thu Dec 26 14:30:00 MSK 2024
```

## Два синтаксиса

### Современный: $(команда)

```bash
$ NOW=$(date)
$ echo $NOW
Thu Dec 26 14:30:00 MSK 2024

$ FILES=$(ls)
$ echo "Files: $FILES"
```

### Устаревший: `команда` (обратные кавычки)

```bash
$ NOW=`date`
$ echo $NOW
Thu Dec 26 14:30:00 MSK 2024
```

**Рекомендуется использовать `$(...)`** — он легче читается и поддерживает вложенность.

## Преимущества $()

### Вложенность
```bash
# С $() — легко
$ echo "Config: $(cat $(which python3).cfg 2>/dev/null || echo 'not found')"

# С обратными кавычками — сложно
$ echo "Config: `cat \`which python3\`.cfg 2>/dev/null || echo 'not found'`"
```

### Читаемость
```bash
# Понятно
$ RESULT=$(ls -la | grep "pattern" | wc -l)

# Менее понятно
$ RESULT=`ls -la | grep "pattern" | wc -l`
```

## Практические примеры

### Сохранение вывода в переменную
```bash
$ HOSTNAME=$(hostname)
$ USER=$(whoami)
$ echo "Logged in as $USER on $HOSTNAME"
```

### Использование в командах
```bash
$ mkdir "backup-$(date +%Y%m%d)"
$ touch "file-$(date +%H%M%S).txt"
```

### В условиях
```bash
if [ "$(whoami)" = "root" ]; then
    echo "Running as root"
fi

FILE_COUNT=$(ls | wc -l)
if [ "$FILE_COUNT" -gt 100 ]; then
    echo "Too many files!"
fi
```

### С циклами
```bash
for file in $(find . -name "*.txt"); do
    echo "Processing: $file"
done
```

### В аргументах
```bash
$ grep "error" $(find /var/log -name "*.log")
```

## Вложенные подстановки

```bash
# Текущая директория git-репозитория
$ echo $(basename $(git rev-parse --show-toplevel))

# Получить версию Python
$ echo "Python version: $(python3 --version | cut -d' ' -f2)"
```

## Подстановка с конвейерами

```bash
$ TOTAL=$(cat sales.csv | awk -F',' '{sum+=$3} END {print sum}')

$ ERROR_COUNT=$(grep -c "ERROR" /var/log/app.log)

$ TOP_PROCESS=$(ps aux --sort=-%mem | head -2 | tail -1)
```

## Присваивание нескольких переменных

### Через массивы
```bash
$ read -r WIDTH HEIGHT <<< $(identify -format "%w %h" image.jpg)
$ echo "Width: $WIDTH, Height: $HEIGHT"
```

### Через eval (осторожно!)
```bash
$ eval $(stat --format='SIZE=%s MODE=%a' file.txt)
$ echo "Size: $SIZE, Mode: $MODE"
```

## Арифметические операции

```bash
$ LINES=$(wc -l < file.txt)
$ DOUBLED=$((LINES * 2))
$ echo "Double: $DOUBLED"

$ echo "Files count: $(( $(ls | wc -l) + 1 ))"
```

## Многострочный вывод

```bash
$ OUTPUT=$(cat << EOF
Line 1
Line 2
Line 3
EOF
)
$ echo "$OUTPUT"       # с кавычками — сохраняет переносы
```

## Обработка ошибок

### Сохранение кода возврата
```bash
$ RESULT=$(some_command)
$ STATUS=$?            # код возврата ПОСЛЕ подстановки

if [ $STATUS -eq 0 ]; then
    echo "Success: $RESULT"
else
    echo "Failed with code $STATUS"
fi
```

### С перенаправлением stderr
```bash
# Захватить и stdout, и stderr
$ OUTPUT=$(command 2>&1)

# Только stdout, stderr в файл
$ OUTPUT=$(command 2>errors.log)

# Только stdout, stderr игнорировать
$ OUTPUT=$(command 2>/dev/null)
```

## Примеры скриптов

### Создание уникального имени файла
```bash
#!/bin/bash
UNIQUE_NAME="file-$(date +%Y%m%d-%H%M%S)-$$"
touch "$UNIQUE_NAME.txt"
```

### Версионирование
```bash
#!/bin/bash
VERSION=$(git describe --tags 2>/dev/null || echo "dev")
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "Version: $VERSION, Built: $BUILD_DATE"
```

### Проверка зависимостей
```bash
#!/bin/bash
for cmd in git docker python3; do
    if ! LOCATION=$(command -v "$cmd" 2>/dev/null); then
        echo "Error: $cmd not found"
        exit 1
    fi
    echo "$cmd found at $LOCATION"
done
```

## Подводные камни

### Пробелы в выводе
```bash
# Плохо — word splitting
$ for f in $(ls); do echo "$f"; done

# Лучше
$ for f in *; do echo "$f"; done
```

### Потеря переносов строк
```bash
$ VAR=$(echo -e "one\ntwo\nthree")
$ echo $VAR          # one two three (переносы потеряны)
$ echo "$VAR"        # сохраняет переносы
```

### Код возврата в конвейере
```bash
# Код возврата — от последней команды
$ RESULT=$(false | cat)
$ echo $?            # 0 (от cat)

$ set -o pipefail    # учитывать все команды
$ RESULT=$(false | cat)
$ echo $?            # 1 (от false)
```

## Советы

1. **Используйте `$(...)`** вместо обратных кавычек
2. **Кавычки вокруг `"$(...)"`** сохраняют пробелы и переносы
3. **Проверяйте `$?`** сразу после подстановки
4. **Избегайте `$(ls)`** — используйте глоббинг `*`
5. **Вложенность** — мощный инструмент, но не злоупотребляйте
