# Перенаправление ошибок (stderr)

[prev: 02-redirecting-output](./02-redirecting-output.md) | [next: 04-cat](./04-cat.md)

---
## Операторы для stderr

| Оператор | Действие |
|----------|----------|
| `2>` | Перенаправить stderr в файл (перезаписать) |
| `2>>` | Перенаправить stderr в файл (добавить) |
| `2>&1` | Перенаправить stderr в stdout |
| `&>` | Перенаправить и stdout, и stderr (bash) |
| `&>>` | Добавить и stdout, и stderr (bash) |

## Перенаправление stderr в файл

### Перезапись (2>)
```bash
$ ls nonexistent 2> errors.txt
$ cat errors.txt
ls: cannot access 'nonexistent': No such file or directory
```

### Добавление (2>>)
```bash
$ ls nonexistent 2>> errors.log
$ ls another_missing 2>> errors.log
$ cat errors.log
ls: cannot access 'nonexistent': No such file or directory
ls: cannot access 'another_missing': No such file or directory
```

## Разделение stdout и stderr

```bash
$ ls existing.txt nonexistent.txt > stdout.txt 2> stderr.txt
$ cat stdout.txt
existing.txt
$ cat stderr.txt
ls: cannot access 'nonexistent.txt': No such file or directory
```

Это полезно для:
- Логирования ошибок отдельно от нормального вывода
- Обработки ошибок без влияния на данные

## Подавление ошибок

Отправить stderr в /dev/null:

```bash
$ ls nonexistent 2> /dev/null
# Ничего не выведется

$ find / -name "*.conf" 2> /dev/null
# Поиск без сообщений "Permission denied"
```

## Объединение stdout и stderr

### Способ 1: 2>&1
```bash
$ command > output.txt 2>&1
# stdout → output.txt
# stderr → stdout → output.txt
```

**Важно**: порядок имеет значение!
```bash
$ command > file 2>&1    # правильно: stderr идёт в file
$ command 2>&1 > file    # неправильно: stderr идёт в терминал
```

### Способ 2: &> (bash)
```bash
$ command &> output.txt
# Эквивалентно: command > output.txt 2>&1
```

### Способ 3: &>> (bash)
```bash
$ command &>> output.txt
# Добавляет оба потока в файл
```

## Перенаправление stderr в stdout

Иногда нужно обработать stderr вместе с stdout в пайплайне:

```bash
$ command 2>&1 | grep "error"
# Теперь grep видит и stdout, и stderr
```

Или с `|&` (bash 4+):
```bash
$ command |& grep "error"
# Эквивалентно: command 2>&1 | grep "error"
```

## Перенаправление stdout в stderr

```bash
$ echo "Warning message" >&2
```

Полезно в скриптах для вывода предупреждений:
```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "Usage: $0 filename" >&2
    exit 1
fi
```

## Практические примеры

### Логирование с разделением потоков
```bash
#!/bin/bash
{
    echo "Starting backup..."
    tar -czf backup.tar.gz /data
    echo "Backup complete"
} > backup.log 2> backup-errors.log
```

### Запись всего в один лог
```bash
$ ./script.sh > script.log 2>&1
# или
$ ./script.sh &> script.log
```

### Показать только ошибки
```bash
$ make > /dev/null
# stdout отброшен, ошибки видны
```

### Показать только успешный вывод
```bash
$ make 2> /dev/null
# ошибки отброшены, stdout виден
```

### Поиск без ошибок доступа
```bash
$ find / -name "*.log" 2> /dev/null
```

### Проверка успешности команды
```bash
$ if command 2> /dev/null; then
    echo "Success"
else
    echo "Failed"
fi
```

## Сложные перенаправления

### Поменять местами stdout и stderr
```bash
$ command 3>&1 1>&2 2>&3 3>&-
# 3 = копия stdout
# 1 → stderr
# 2 → 3 (бывший stdout)
# закрыть 3
```

### Разные действия для каждого потока
```bash
$ command 2>&1 1>/dev/null | process_errors
# stdout отброшен, stderr обрабатывается
```

### Сохранить и вывести на экран
```bash
$ command 2>&1 | tee output.log
# stdout и stderr на экран и в файл
```

## exec для глобального перенаправления

В скриптах можно перенаправить все последующие команды:

```bash
#!/bin/bash
exec 2> errors.log   # все ошибки с этого момента в файл

echo "Working..."    # stdout на экран
ls nonexistent       # ошибка в errors.log
cd /nonexistent      # ошибка в errors.log
```

### Логирование всего скрипта
```bash
#!/bin/bash
exec > >(tee -a script.log) 2>&1

echo "This goes to screen and log"
ls nonexistent  # ошибки тоже в лог
```

## Важные замечания

1. **Файл создаётся до выполнения команды**:
   ```bash
   $ cat file 2> file   # осторожно!
   ```

2. **Порядок операторов важен**:
   ```bash
   $ cmd > log 2>&1    # всё в log
   $ cmd 2>&1 > log    # только stdout в log, stderr на экран
   ```

3. **&> не работает в sh**, только в bash:
   ```bash
   # Для совместимости используйте:
   $ command > file 2>&1
   ```

4. **Номера дескрипторов пишутся слитно** с оператором:
   ```bash
   $ command 2> file    # правильно
   $ command 2 > file   # неправильно (2 — аргумент)
   ```

---

[prev: 02-redirecting-output](./02-redirecting-output.md) | [next: 04-cat](./04-cat.md)
