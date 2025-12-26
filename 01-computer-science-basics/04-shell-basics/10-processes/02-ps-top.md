# Команды ps и top

## Команда ps

**ps** (process status) — показывает снимок текущих процессов.

### Синтаксис
```bash
ps [опции]
```

### Базовое использование

```bash
$ ps
PID TTY          TIME CMD
1234 pts/0    00:00:00 bash
5678 pts/0    00:00:00 ps
```

## Стили опций ps

ps поддерживает три стиля опций:
- **Unix (с дефисом)**: `ps -ef`
- **BSD (без дефиса)**: `ps aux`
- **GNU (два дефиса)**: `ps --help`

### ps aux — все процессы (BSD стиль)

```bash
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 169584 11896 ?        Ss   Dec25   0:03 /sbin/init
root         2  0.0  0.0      0     0 ?        S    Dec25   0:00 [kthreadd]
alice     1234  0.1  0.5 123456 45678 pts/0    Ss   10:00   0:01 bash
```

Колонки:
| Колонка | Описание |
|---------|----------|
| USER | Владелец процесса |
| PID | ID процесса |
| %CPU | Использование CPU |
| %MEM | Использование памяти |
| VSZ | Виртуальная память (KB) |
| RSS | Резидентная память (KB) |
| TTY | Терминал |
| STAT | Состояние |
| START | Время запуска |
| TIME | CPU время |
| COMMAND | Команда |

### ps -ef — все процессы (Unix стиль)

```bash
$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Dec25 ?        00:00:03 /sbin/init
alice     1234  1230  0 10:00 pts/0    00:00:01 bash
```

## Полезные варианты ps

### Конкретный процесс
```bash
$ ps -p 1234
$ ps -p 1234 -o pid,user,cmd,stat
```

### Процессы пользователя
```bash
$ ps -u alice
$ ps -U root -u root
```

### С деревом
```bash
$ ps -ef --forest
root         1     0  0 Dec25 ?        /sbin/init
root      1230     1  0 10:00 ?         \_ sshd
alice     1234  1230  0 10:00 pts/0         \_ bash
alice     5678  1234  0 10:05 pts/0             \_ vim
```

### Сортировка
```bash
$ ps aux --sort=-%mem | head    # по памяти (убывание)
$ ps aux --sort=-%cpu | head    # по CPU (убывание)
$ ps aux --sort=start_time      # по времени запуска
```

### Выбор колонок
```bash
$ ps -eo pid,ppid,user,stat,cmd
$ ps -eo pid,pcpu,pmem,cmd --sort=-pcpu | head -10
```

## Состояния процессов (STAT)

| Символ | Значение |
|--------|----------|
| R | Running (выполняется) |
| S | Sleeping (ждёт событие) |
| D | Disk sleep (ждёт I/O) |
| T | Stopped (остановлен) |
| Z | Zombie |

Дополнительные символы:
| Символ | Значение |
|--------|----------|
| s | Лидер сессии |
| l | Многопоточный |
| + | Foreground группа |
| < | Высокий приоритет |
| N | Низкий приоритет |

```bash
$ ps aux
# STAT колонка:
# Ss   — sleeping, session leader
# S+   — sleeping, foreground
# R+   — running, foreground
```

## Команда top

**top** — интерактивный монитор процессов в реальном времени.

### Запуск
```bash
$ top
```

### Интерфейс top

```
top - 14:30:00 up 2 days,  3:22,  2 users,  load average: 0.15, 0.10, 0.05
Tasks: 200 total,   1 running, 199 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 93.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  16000.0 total,   8000.0 free,   4000.0 used,   4000.0 buff/cache
MiB Swap:   2000.0 total,   2000.0 free,      0.0 used.  11000.0 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 alice     20   0  123456  45678  12345 S   5.0   0.3   0:30.00 firefox
```

### Горячие клавиши в top

| Клавиша | Действие |
|---------|----------|
| `q` | Выход |
| `h` или `?` | Справка |
| `k` | Kill процесс |
| `r` | Renice (изменить приоритет) |
| `u` | Фильтр по пользователю |
| `M` | Сортировка по памяти |
| `P` | Сортировка по CPU |
| `T` | Сортировка по времени |
| `c` | Показать полный путь |
| `1` | Показать все CPU |
| `m` | Переключить отображение памяти |
| `t` | Переключить отображение CPU |

### Опции top

```bash
$ top -u alice              # только процессы alice
$ top -p 1234,5678          # только указанные PID
$ top -n 5                  # 5 обновлений и выход
$ top -b > log.txt          # batch mode (для скриптов)
```

## Альтернативы top

### htop — улучшенный top

```bash
$ htop
```

Преимущества:
- Цветной интерфейс
- Мышь работает
- Горизонтальная и вертикальная прокрутка
- Фильтрация и поиск
- Древовидный вид

### btop — современный монитор

```bash
$ btop
```

### glances — обзор системы

```bash
$ glances
```

## Практические примеры

### Найти "тяжёлые" процессы
```bash
# По CPU
$ ps aux --sort=-%cpu | head -10

# По памяти
$ ps aux --sort=-%mem | head -10
```

### Найти процесс по имени
```bash
$ ps aux | grep nginx
$ pgrep -a nginx
$ pidof nginx
```

### Мониторинг конкретного процесса
```bash
$ top -p $(pgrep -d',' nginx)
```

### Найти zombie процессы
```bash
$ ps aux | awk '$8 ~ /Z/ { print }'
```

### Найти долго работающие процессы
```bash
$ ps -eo pid,etime,cmd --sort=-etime | head
```

### Подсчёт процессов
```bash
$ ps aux | wc -l
$ ps -e --no-headers | wc -l
```
