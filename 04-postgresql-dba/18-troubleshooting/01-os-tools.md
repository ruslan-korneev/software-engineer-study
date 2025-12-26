# Инструменты операционной системы для диагностики PostgreSQL

При диагностике проблем с производительностью PostgreSQL важно понимать, как база данных взаимодействует с операционной системой. Инструменты ОС помогают выявить узкие места на уровне CPU, памяти, дисковой подсистемы и сети.

## top и htop

### top - базовый мониторинг процессов

```bash
# Запуск top с фильтрацией по PostgreSQL
top -p $(pgrep -d',' postgres)

# Полезные клавиши в top:
# 1 - показать загрузку каждого ядра CPU
# M - сортировка по памяти
# P - сортировка по CPU
# c - показать полную командную строку
```

**На что обращать внимание:**
- **%CPU** - высокая загрузка может указывать на тяжёлые запросы
- **%MEM** - потребление памяти процессами PostgreSQL
- **RES** - резидентная память (реально используемая)
- **VIRT** - виртуальная память (включая shared buffers)

### htop - улучшенная версия top

```bash
# Установка
sudo apt install htop  # Debian/Ubuntu
sudo yum install htop  # CentOS/RHEL

# Фильтрация по postgres
htop -p $(pgrep -d',' postgres)
```

**Преимущества htop:**
- Цветовая индикация загрузки
- Древовидное отображение процессов (F5)
- Интерактивный поиск (F3) и фильтрация (F4)
- Отображение использования CPU и памяти в виде графиков

## iostat - мониторинг дисковой подсистемы

```bash
# Установка (входит в пакет sysstat)
sudo apt install sysstat

# Базовый вывод с интервалом 2 секунды
iostat -x 2

# Мониторинг конкретного диска
iostat -x /dev/sda 2

# Вывод в мегабайтах
iostat -xm 2
```

**Ключевые метрики для PostgreSQL:**

| Метрика | Описание | Проблемные значения |
|---------|----------|---------------------|
| %util | Утилизация диска | >80% - перегрузка |
| await | Среднее время ожидания I/O (мс) | >20мс для SSD, >50мс для HDD |
| r/s, w/s | Операции чтения/записи в секунду | Зависит от диска |
| rkB/s, wkB/s | Скорость чтения/записи | - |
| avgqu-sz | Средняя длина очереди | >1 указывает на нагрузку |

```bash
# Пример вывода с анализом
Device     r/s    w/s    rkB/s   wkB/s  await  %util
sda       150.0  200.0  6000.0  8000.0   5.2   45.0
# Здесь всё в норме: await низкий, util < 80%
```

## vmstat - статистика виртуальной памяти

```bash
# Вывод каждые 2 секунды
vmstat 2

# Вывод с отметками времени
vmstat -t 2

# Расширенный вывод
vmstat -w 2
```

**Важные колонки:**

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 2048000  1024 4096000   0    0    50   100  500 1000 10  5 84  1  0
```

- **r** - процессы в очереди на выполнение (высокое значение = нехватка CPU)
- **b** - процессы в ожидании I/O (высокое значение = проблемы с диском)
- **swpd** - использование swap (должно быть 0 или минимально)
- **si/so** - swap in/out (активный swap = критическая проблема)
- **wa** - CPU wait (ожидание I/O, высокое значение = узкое место в диске)

## sar - System Activity Reporter

```bash
# Включение сбора статистики
sudo systemctl enable sysstat
sudo systemctl start sysstat

# CPU статистика за сегодня
sar -u

# Статистика памяти
sar -r

# Дисковая статистика
sar -d

# Сетевая статистика
sar -n DEV

# Статистика за конкретную дату
sar -u -f /var/log/sysstat/sa20
```

**Примеры анализа:**

```bash
# Найти пики загрузки CPU
sar -u | awk '$8 < 10 {print}'  # Показать когда idle < 10%

# Анализ использования памяти
sar -r | grep -v Average | tail -20

# История дисковой активности
sar -d -p | grep sda
```

## strace - трассировка системных вызовов

```bash
# Трассировка конкретного процесса PostgreSQL
sudo strace -p <pid> -e trace=read,write,open

# Подсчёт системных вызовов
sudo strace -c -p <pid>

# Трассировка с временными метками
sudo strace -tt -p <pid>

# Запись в файл
sudo strace -o /tmp/postgres_trace.log -p <pid>
```

**Типичные сценарии использования:**

```bash
# Найти PID backend-процесса PostgreSQL
SELECT pg_backend_pid();  -- в psql

# Посмотреть, что делает "зависший" процесс
sudo strace -p 12345 -e trace=read,write,poll,select

# Анализ работы с файлами
sudo strace -p 12345 -e trace=file
```

**Интерпретация вывода:**
```
read(3, "SELECT * FROM users"..., 8192) = 25
# Чтение 25 байт из файлового дескриптора 3

write(8, "\x01\x00\x00\x00..."..., 4096) = 4096
# Запись 4096 байт в файловый дескриптор 8
```

## perf - профилировщик производительности Linux

```bash
# Установка
sudo apt install linux-tools-common linux-tools-$(uname -r)

# Запись профиля CPU для процесса
sudo perf record -p <pid> -g -- sleep 30

# Анализ записанного профиля
sudo perf report

# Статистика производительности в реальном времени
sudo perf top -p <pid>

# Запись для всех процессов postgres
sudo perf record -g -p $(pgrep -d',' postgres) -- sleep 60
```

**Flame Graphs с perf:**

```bash
# Генерация данных для flame graph
sudo perf record -F 99 -p <pid> -g -- sleep 60
sudo perf script > out.perf

# Использование FlameGraph (github.com/brendangregg/FlameGraph)
./stackcollapse-perf.pl out.perf > out.folded
./flamegraph.pl out.folded > flamegraph.svg
```

## Комплексный скрипт мониторинга

```bash
#!/bin/bash
# postgres_monitor.sh - комплексный мониторинг PostgreSQL

echo "=== CPU и процессы ==="
ps aux | grep postgres | head -5

echo -e "\n=== Память ==="
free -h

echo -e "\n=== Swap активность ==="
vmstat 1 3 | tail -2

echo -e "\n=== Дисковая активность ==="
iostat -x 1 3 | grep -A1 Device | tail -2

echo -e "\n=== Открытые файлы PostgreSQL ==="
lsof -c postgres | wc -l

echo -e "\n=== Сетевые соединения ==="
ss -s | grep TCP
```

## Типичные проблемы и диагностика

### Высокая загрузка CPU
```bash
# 1. Найти процесс PostgreSQL с высокой загрузкой
top -p $(pgrep -d',' postgres)

# 2. Определить PID и найти запрос
sudo strace -p <pid> -c -w
# или через pg_stat_activity
```

### Проблемы с диском
```bash
# 1. Проверить утилизацию
iostat -x 2 5

# 2. Найти процессы с активным I/O
iotop -P

# 3. Проверить очередь записи WAL
ls -la /var/lib/postgresql/*/main/pg_wal/
```

### Нехватка памяти
```bash
# 1. Общая картина
free -h
vmstat 2 5

# 2. Память процессов PostgreSQL
ps aux --sort=-%mem | grep postgres | head -10

# 3. Проверить OOM killer
dmesg | grep -i "out of memory"
```

## Полезные алиасы для bash

```bash
# Добавить в ~/.bashrc
alias pgtop='top -p $(pgrep -d"," postgres)'
alias pgmem='ps aux --sort=-%mem | grep postgres | head'
alias pgdisk='iostat -xm 2 | grep -A1 Device'
alias pgnet='ss -tnp | grep postgres'
```

## Заключение

Инструменты операционной системы незаменимы для:
- Первичной диагностики проблем производительности
- Мониторинга ресурсов в реальном времени
- Анализа исторических данных (sar)
- Глубокого профилирования (strace, perf)

Рекомендуется использовать их в комбинации с инструментами PostgreSQL (pg_stat_* представления) для полной картины.
