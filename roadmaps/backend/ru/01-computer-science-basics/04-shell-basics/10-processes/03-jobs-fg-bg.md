# Управление заданиями: jobs, fg, bg

## Foreground и Background

- **Foreground** — процесс занимает терминал, получает ввод
- **Background** — процесс работает в фоне, терминал свободен

## Запуск в фоне (&)

Добавьте `&` в конце команды:

```bash
$ sleep 100 &
[1] 12345

# [1] — номер задания
# 12345 — PID
```

Терминал сразу доступен для новых команд.

## Команда jobs

Показывает фоновые задания текущего shell:

```bash
$ sleep 100 &
[1] 12345
$ sleep 200 &
[2] 12346

$ jobs
[1]-  Running                 sleep 100 &
[2]+  Running                 sleep 200 &
```

### Обозначения

| Символ | Значение |
|--------|----------|
| `+` | Текущее задание (последнее) |
| `-` | Предыдущее задание |

### Опции jobs

```bash
$ jobs -l           # с PID
[1]- 12345 Running  sleep 100 &
[2]+ 12346 Running  sleep 200 &

$ jobs -p           # только PID
12345
12346

$ jobs -r           # только запущенные
$ jobs -s           # только остановленные
```

## Приостановка процесса (Ctrl+Z)

Ctrl+Z приостанавливает foreground процесс:

```bash
$ vim file.txt
# работаем в vim...
^Z                  # Ctrl+Z
[1]+  Stopped       vim file.txt
$                   # терминал свободен
```

Процесс приостановлен (SIGTSTP), но не завершён.

## Команда bg — продолжить в фоне

```bash
$ sleep 100
^Z
[1]+  Stopped       sleep 100

$ bg
[1]+ sleep 100 &

$ jobs
[1]+  Running       sleep 100 &
```

С указанием задания:
```bash
$ bg %1             # задание 1
$ bg %2             # задание 2
```

## Команда fg — вернуть на передний план

```bash
$ sleep 100 &
[1] 12345

$ fg
sleep 100           # теперь в foreground
^C                  # можно прервать
```

С указанием задания:
```bash
$ fg %1             # задание 1
$ fg %vim           # задание по имени
```

## Ссылки на задания

| Синтаксис | Значение |
|-----------|----------|
| `%n` | Задание номер n |
| `%+` или `%%` | Текущее задание |
| `%-` | Предыдущее задание |
| `%string` | Задание, начинающееся с string |
| `%?string` | Задание, содержащее string |

```bash
$ fg %1
$ fg %+
$ fg %vim
$ fg %?edit
```

## Практические сценарии

### Запуск долгой операции в фоне

```bash
# Компиляция
$ make -j4 &

# Скачивание
$ wget http://example.com/large-file.zip &

# Резервное копирование
$ tar -czf backup.tar.gz /data &
```

### Несколько фоновых задач

```bash
$ ./script1.sh &
$ ./script2.sh &
$ ./script3.sh &
$ jobs
[1]   Running    ./script1.sh &
[2]-  Running    ./script2.sh &
[3]+  Running    ./script3.sh &
```

### Приостановить и возобновить

```bash
$ vim largefile.txt
^Z
[1]+  Stopped    vim largefile.txt

$ # делаем что-то другое
$ ls /tmp
...

$ fg              # возвращаемся в vim
```

## Команда disown

Отвязать задание от shell (не завершится при выходе):

```bash
$ long-running-task &
[1] 12345

$ disown %1       # отвязать задание 1
$ disown -a       # отвязать все
$ disown -h %1    # не отправлять SIGHUP при закрытии shell
```

## nohup — защита от hangup

Запустить процесс, защищённый от закрытия терминала:

```bash
$ nohup long-task &
nohup: ignoring input and appending output to 'nohup.out'

# Вывод сохраняется в nohup.out
# Процесс продолжит работу после выхода из сессии
```

С перенаправлением:
```bash
$ nohup ./script.sh > output.log 2>&1 &
```

## screen и tmux — альтернатива

Для длительных сессий лучше использовать терминальные мультиплексоры:

```bash
# screen
$ screen -S mySession
$ ./long-task
# Ctrl+A, D — отключиться
$ screen -r mySession  # подключиться обратно

# tmux
$ tmux new -s mySession
$ ./long-task
# Ctrl+B, D — отключиться
$ tmux attach -t mySession
```

## Статусы заданий

| Статус | Описание |
|--------|----------|
| Running | Выполняется |
| Stopped | Приостановлено (Ctrl+Z) |
| Done | Завершено успешно |
| Exit N | Завершено с кодом N |
| Terminated | Прервано сигналом |

```bash
$ sleep 1 &
[1] 12345
$ # через секунду...
[1]+  Done       sleep 1
```

## Важные замечания

### Вывод фоновых процессов

Фоновые процессы всё ещё выводят в терминал:
```bash
$ yes > /dev/null &
$ find / -name "*.txt" &
# Вывод появится в терминале
```

Решение — перенаправить вывод:
```bash
$ find / -name "*.txt" > results.txt 2>&1 &
```

### При закрытии терминала

По умолчанию фоновые задания получат SIGHUP и завершатся:
```bash
$ long-task &
$ exit
# long-task завершится
```

Защита:
```bash
$ nohup long-task &
$ disown %1
# Или используйте screen/tmux
```

## Сводка команд

| Команда | Действие |
|---------|----------|
| `command &` | Запустить в фоне |
| `Ctrl+Z` | Приостановить foreground |
| `jobs` | Список заданий |
| `bg` | Продолжить в фоне |
| `fg` | Вернуть в foreground |
| `disown` | Отвязать от shell |
| `nohup` | Защита от SIGHUP |
