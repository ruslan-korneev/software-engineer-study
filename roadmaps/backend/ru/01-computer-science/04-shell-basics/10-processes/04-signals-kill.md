# Сигналы и команда kill

[prev: 03-jobs-fg-bg](./03-jobs-fg-bg.md) | [next: 05-shutdown](./05-shutdown.md)

---
## Что такое сигналы?

**Сигналы** — механизм межпроцессного взаимодействия в Unix. Это асинхронные уведомления, отправляемые процессам для управления их поведением.

## Основные сигналы

| Номер | Имя | Описание | Действие |
|-------|-----|----------|----------|
| 1 | SIGHUP | Hangup (закрытие терминала) | Завершение/перезагрузка |
| 2 | SIGINT | Interrupt (Ctrl+C) | Завершение |
| 3 | SIGQUIT | Quit (Ctrl+\\) | Завершение с core dump |
| 9 | SIGKILL | Kill (принудительно) | Немедленное завершение |
| 15 | SIGTERM | Terminate | Корректное завершение |
| 18 | SIGCONT | Continue | Продолжить выполнение |
| 19 | SIGSTOP | Stop | Приостановить (нельзя перехватить) |
| 20 | SIGTSTP | Terminal Stop (Ctrl+Z) | Приостановить |

### Полный список сигналов
```bash
$ kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
 5) SIGTRAP      6) SIGABRT      7) SIGBUS       8) SIGFPE
 9) SIGKILL     10) SIGUSR1     11) SIGSEGV     12) SIGUSR2
13) SIGPIPE     14) SIGALRM     15) SIGTERM     ...
```

## Команда kill

**kill** — отправляет сигнал процессу.

### Синтаксис
```bash
kill [сигнал] PID...
```

### Базовое использование

```bash
$ kill 12345              # SIGTERM (по умолчанию)
$ kill -15 12345          # то же самое
$ kill -TERM 12345        # то же самое
$ kill -SIGTERM 12345     # то же самое
```

### Примеры

```bash
# Корректное завершение
$ kill 12345

# Принудительное завершение
$ kill -9 12345
$ kill -KILL 12345

# Приостановить
$ kill -STOP 12345

# Продолжить
$ kill -CONT 12345

# Перезагрузить конфигурацию (для демонов)
$ kill -HUP 12345
```

## SIGTERM vs SIGKILL

### SIGTERM (15) — корректное завершение
- Процесс может перехватить
- Может выполнить очистку (закрыть файлы, сохранить данные)
- Процесс может игнорировать

```bash
$ kill 12345        # даёт процессу шанс завершиться корректно
```

### SIGKILL (9) — принудительное завершение
- Нельзя перехватить или игнорировать
- Немедленное завершение ядром
- Нет шанса на очистку

```bash
$ kill -9 12345     # крайняя мера
```

**Правило**: сначала SIGTERM, ждите, затем SIGKILL если нужно.

## Команда killall

Завершает процессы по имени:

```bash
$ killall firefox
$ killall -9 firefox      # принудительно
$ killall -u alice        # все процессы пользователя
$ killall -i firefox      # интерактивно (с подтверждением)
```

## Команда pkill

Завершает процессы по шаблону:

```bash
$ pkill firefox
$ pkill -9 firefox
$ pkill -f "python script.py"   # по полной командной строке
$ pkill -u alice                # по пользователю
$ pkill -t pts/0                # по терминалу
```

## Команда pgrep

Находит PID по шаблону (без kill):

```bash
$ pgrep firefox
12345
12346

$ pgrep -a firefox              # с командной строкой
12345 /usr/lib/firefox/firefox

$ pgrep -l firefox              # с именем
12345 firefox
```

## Отправка сигналов с клавиатуры

| Комбинация | Сигнал | Действие |
|------------|--------|----------|
| Ctrl+C | SIGINT | Прервать |
| Ctrl+Z | SIGTSTP | Приостановить |
| Ctrl+\\ | SIGQUIT | Завершить с core dump |

## Обработка сигналов в скриптах

```bash
#!/bin/bash

# Установить обработчик
cleanup() {
    echo "Получен сигнал, очистка..."
    rm -f /tmp/myfile
    exit 0
}

trap cleanup SIGTERM SIGINT

# Основной код
while true; do
    echo "Работаю..."
    sleep 1
done
```

### Синтаксис trap
```bash
trap 'команды' СИГНАЛЫ
trap '' SIGINT           # игнорировать сигнал
trap - SIGINT            # сбросить обработчик
```

## Практические сценарии

### Завершить зависший процесс
```bash
$ ps aux | grep firefox
alice 12345 ... firefox

$ kill 12345              # сначала SIGTERM
$ sleep 5                 # ждём
$ kill -9 12345           # если не помогло
```

### Перезагрузить конфигурацию демона
```bash
$ sudo kill -HUP $(pidof nginx)
# или
$ sudo systemctl reload nginx
```

### Завершить все процессы Python
```bash
$ pkill python
$ killall python3
```

### Завершить процессы по шаблону
```bash
$ pkill -f "node server.js"
$ pkill -f "java.*MyApp"
```

### Безопасный скрипт завершения
```bash
#!/bin/bash
PID=$1

if kill -0 "$PID" 2>/dev/null; then
    kill "$PID"
    sleep 3
    if kill -0 "$PID" 2>/dev/null; then
        echo "Процесс не завершился, использую SIGKILL"
        kill -9 "$PID"
    fi
else
    echo "Процесс $PID не существует"
fi
```

## Специальные сигналы

### SIGUSR1 и SIGUSR2
Пользовательские сигналы для своих целей:
```bash
$ kill -USR1 12345        # приложение может реагировать по-своему
```

### SIGHUP для демонов
Традиционно означает "перечитать конфигурацию":
```bash
$ sudo kill -HUP $(cat /var/run/nginx.pid)
```

## Отладка

### Проверить, жив ли процесс
```bash
$ kill -0 12345           # не посылает сигнал, проверяет существование
$ echo $?                 # 0 = существует, 1 = нет
```

### Список сигналов
```bash
$ kill -l
$ kill -l TERM            # номер сигнала
15
```

## Советы

1. **Всегда начинайте с SIGTERM** — дайте процессу завершиться корректно
2. **SIGKILL — крайняя мера** — может повредить данные
3. **Используйте pkill -f** для сложных шаблонов
4. **trap в скриптах** для корректного завершения
5. **kill -0** для проверки существования процесса

---

[prev: 03-jobs-fg-bg](./03-jobs-fg-bg.md) | [next: 05-shutdown](./05-shutdown.md)
