# Стандартные потоки: stdin, stdout, stderr

[prev: 03-alias](../05-working-with-commands/03-alias.md) | [next: 02-redirecting-output](./02-redirecting-output.md)

---
## Концепция потоков

В Unix/Linux каждая программа имеет три стандартных потока данных:

| Поток | Номер (FD) | Назначение |
|-------|------------|------------|
| **stdin** | 0 | Стандартный ввод |
| **stdout** | 1 | Стандартный вывод |
| **stderr** | 2 | Стандартный вывод ошибок |

FD — File Descriptor (файловый дескриптор), числовой идентификатор потока.

## Визуализация

```
              ┌──────────────┐
  stdin  →────│              │────→  stdout
   (0)        │   Программа  │        (1)
              │              │────→  stderr
              └──────────────┘        (2)
```

## stdin (стандартный ввод)

**stdin** — поток данных, которые программа получает на вход.

По умолчанию stdin связан с клавиатурой:

```bash
$ cat                     # ждёт ввод с клавиатуры
Hello, World!             # вводим текст
Hello, World!             # cat выводит его
^D                        # Ctrl+D завершает ввод
```

stdin можно перенаправить из файла:
```bash
$ cat < file.txt          # stdin из файла
```

## stdout (стандартный вывод)

**stdout** — поток для нормального вывода программы.

По умолчанию stdout выводится на экран (терминал):

```bash
$ echo "Hello"
Hello                     # вывод в stdout → экран

$ ls
file1.txt  file2.txt      # вывод в stdout → экран
```

stdout можно перенаправить в файл:
```bash
$ echo "Hello" > file.txt
$ ls > listing.txt
```

## stderr (стандартный вывод ошибок)

**stderr** — отдельный поток для сообщений об ошибках.

```bash
$ ls nonexistent
ls: cannot access 'nonexistent': No such file or directory
#   ↑ это stderr
```

Почему stderr отделён от stdout?

```bash
# Ошибки видны на экране, даже если stdout перенаправлен
$ ls nonexistent > output.txt
ls: cannot access 'nonexistent': No such file or directory
# Ошибка появилась на экране, файл output.txt пустой
```

## Практические примеры

### Программа с обоими выводами
```bash
$ ls existing_file nonexistent_file
existing_file                                         # stdout
ls: cannot access 'nonexistent_file': No such file    # stderr
```

### Разделение потоков
```bash
$ ls existing nonexistent > stdout.txt 2> stderr.txt
$ cat stdout.txt
existing
$ cat stderr.txt
ls: cannot access 'nonexistent': No such file or directory
```

## Файловые дескрипторы в деталях

Дескрипторы — это числа, по которым процесс обращается к потокам:

```bash
# Перенаправление по номеру дескриптора
$ command 1> stdout.txt    # stdout в файл
$ command 2> stderr.txt    # stderr в файл
$ command 0< input.txt     # stdin из файла
```

## Проверка потоков

### Вывод в конкретный поток (в скриптах)
```bash
#!/bin/bash
echo "Это stdout"          # в stdout (1)
echo "Это stderr" >&2      # в stderr (2)
```

### Программа для демонстрации
```bash
#!/bin/bash
# test-streams.sh
echo "Normal output (stdout)"
echo "Error output (stderr)" >&2
```

```bash
$ ./test-streams.sh
Normal output (stdout)
Error output (stderr)

$ ./test-streams.sh > output.txt
Error output (stderr)          # stderr на экране

$ ./test-streams.sh 2> errors.txt
Normal output (stdout)         # stdout на экране
```

## Терминал как устройство

stdin, stdout и stderr по умолчанию связаны с терминалом:

```bash
$ tty                          # какой терминал
/dev/pts/0

$ ls -l /dev/stdin /dev/stdout /dev/stderr
lrwxrwxrwx 1 root root 15 /dev/stdin -> /proc/self/fd/0
lrwxrwxrwx 1 root root 15 /dev/stdout -> /proc/self/fd/1
lrwxrwxrwx 1 root root 15 /dev/stderr -> /proc/self/fd/2
```

## Интерактивный vs неинтерактивный режим

Некоторые программы ведут себя по-разному в зависимости от того, куда идёт вывод:

```bash
$ ls
file1  file2  file3        # цветной вывод в столбик

$ ls | cat
file1
file2
file3                       # простой вывод, нет цветов
```

Программа `ls` определяет, что stdout — не терминал, и меняет формат вывода.

### Проверка типа потока
```bash
if [ -t 1 ]; then
    echo "stdout — терминал"
else
    echo "stdout — файл или pipe"
fi
```

## Важность разделения потоков

1. **Логирование**: ошибки и вывод можно записывать в разные файлы
2. **Обработка**: stdout можно передать дальше, сохранив ошибки видимыми
3. **Фильтрация**: можно отбросить один поток, сохранив другой

```bash
# Только успешный вывод
$ command 2>/dev/null

# Только ошибки
$ command 2>&1 >/dev/null

# Всё в один файл
$ command > log.txt 2>&1
```

---

[prev: 03-alias](../05-working-with-commands/03-alias.md) | [next: 02-redirecting-output](./02-redirecting-output.md)
