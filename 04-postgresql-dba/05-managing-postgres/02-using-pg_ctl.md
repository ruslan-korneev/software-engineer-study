# Управление PostgreSQL с помощью pg_ctl

## Введение

**pg_ctl** — это официальная утилита PostgreSQL для управления сервером базы данных. Она предоставляет низкоуровневый контроль над процессом postgres и работает на всех платформах, где установлен PostgreSQL (Linux, Windows, macOS, BSD).

pg_ctl особенно полезен в следующих случаях:
- Установка PostgreSQL из исходников (без пакетного менеджера)
- Среды разработки и тестирования
- Кастомные deployment-сценарии
- Системы без systemd (старые Linux, BSD, macOS)

## Основные концепции

### Переменные окружения

pg_ctl использует несколько важных переменных окружения:

```bash
# Директория с данными кластера (обязательно указать)
export PGDATA=/var/lib/postgresql/15/main

# Альтернатива — использовать флаг -D
pg_ctl -D /var/lib/postgresql/15/main status
```

### Расположение pg_ctl

```bash
# PostgreSQL из пакетов (Debian/Ubuntu)
/usr/lib/postgresql/15/bin/pg_ctl

# PostgreSQL из пакетов (RHEL/CentOS)
/usr/pgsql-15/bin/pg_ctl

# Сборка из исходников
/usr/local/pgsql/bin/pg_ctl

# Homebrew на macOS
/usr/local/opt/postgresql@15/bin/pg_ctl
```

## Основные команды

### Запуск сервера

```bash
# Базовый запуск
pg_ctl -D /var/lib/postgresql/15/main start

# Запуск с логированием в файл
pg_ctl -D /var/lib/postgresql/15/main -l /var/log/postgresql/postgresql.log start

# Запуск с дополнительными параметрами postgres
pg_ctl -D /var/lib/postgresql/15/main -o "-p 5433" start

# Запуск в фоновом режиме (по умолчанию) или на переднем плане
pg_ctl -D /var/lib/postgresql/15/main start  # фоновый
pg_ctl -D /var/lib/postgresql/15/main start -w  # ожидать готовности

# Запуск с таймаутом ожидания (в секундах)
pg_ctl -D /var/lib/postgresql/15/main -t 60 -w start
```

### Остановка сервера

PostgreSQL поддерживает три режима остановки:

```bash
# Smart shutdown (по умолчанию)
# Ждёт завершения всех клиентских соединений
pg_ctl -D /var/lib/postgresql/15/main stop -m smart

# Fast shutdown (рекомендуется для production)
# Откатывает незавершённые транзакции и отключает клиентов
pg_ctl -D /var/lib/postgresql/15/main stop -m fast

# Immediate shutdown (аварийный)
# Немедленная остановка без корректного завершения
# Требует восстановления при следующем запуске
pg_ctl -D /var/lib/postgresql/15/main stop -m immediate

# С ожиданием завершения
pg_ctl -D /var/lib/postgresql/15/main stop -m fast -w
```

### Перезапуск сервера

```bash
# Перезапуск с режимом fast
pg_ctl -D /var/lib/postgresql/15/main restart -m fast

# Перезапуск с новыми параметрами
pg_ctl -D /var/lib/postgresql/15/main restart -m fast -o "-c shared_buffers=512MB"
```

### Перезагрузка конфигурации

```bash
# Перечитать postgresql.conf и pg_hba.conf без перезапуска
pg_ctl -D /var/lib/postgresql/15/main reload
```

Эквивалент SQL команды:
```sql
SELECT pg_reload_conf();
```

### Проверка статуса

```bash
# Проверить, запущен ли сервер
pg_ctl -D /var/lib/postgresql/15/main status
```

Пример вывода:
```
pg_ctl: server is running (PID: 1234)
/usr/lib/postgresql/15/bin/postgres "-D" "/var/lib/postgresql/15/main"
```

## Инициализация кластера

```bash
# Создание нового кластера базы данных
pg_ctl -D /var/lib/postgresql/15/new_cluster initdb

# С дополнительными параметрами initdb
pg_ctl -D /var/lib/postgresql/15/new_cluster initdb -o "--encoding=UTF8 --locale=ru_RU.UTF-8"

# Явное указание суперпользователя
pg_ctl -D /var/lib/postgresql/15/new_cluster initdb -o "--username=postgres --pwprompt"
```

## Promote реплики

```bash
# Превращение standby-сервера в primary (master)
pg_ctl -D /var/lib/postgresql/15/main promote

# С ожиданием завершения
pg_ctl -D /var/lib/postgresql/15/main promote -w
```

Это важная операция при failover — переключении на резервный сервер.

## Регистрация как службы Windows

На Windows pg_ctl может управлять службами:

```cmd
REM Регистрация как служба Windows
pg_ctl register -D "C:\Program Files\PostgreSQL\15\data" -N "PostgreSQL15"

REM Удаление службы
pg_ctl unregister -N "PostgreSQL15"
```

## Расширенные параметры

### Параметр -o (опции postgres)

Передача параметров непосредственно процессу postgres:

```bash
# Изменение порта
pg_ctl -D /var/lib/postgresql/15/main start -o "-p 5433"

# Несколько параметров
pg_ctl -D /var/lib/postgresql/15/main start -o "-p 5433 -c log_connections=on"

# Изменение настроек памяти
pg_ctl -D /var/lib/postgresql/15/main start -o "-c shared_buffers=1GB -c work_mem=256MB"

# Включение режима отладки
pg_ctl -D /var/lib/postgresql/15/main start -o "-d 2"
```

### Параметр -w и -W

```bash
# -w: ожидать завершения операции (wait)
pg_ctl -D /var/lib/postgresql/15/main start -w

# -W: не ожидать завершения (no-wait)
pg_ctl -D /var/lib/postgresql/15/main start -W
```

### Параметр -t (timeout)

```bash
# Установка таймаута ожидания в секундах (по умолчанию 60)
pg_ctl -D /var/lib/postgresql/15/main -t 120 -w start
```

### Параметр -s (silent)

```bash
# Минимальный вывод (только ошибки)
pg_ctl -D /var/lib/postgresql/15/main -s start
```

## Практические сценарии

### Скрипт запуска для разработки

```bash
#!/bin/bash

PGDATA="/home/developer/postgres/data"
PGLOG="/home/developer/postgres/postgres.log"
PGPORT=5432

case "$1" in
    start)
        pg_ctl -D "$PGDATA" -l "$PGLOG" -o "-p $PGPORT" start
        ;;
    stop)
        pg_ctl -D "$PGDATA" stop -m fast
        ;;
    restart)
        pg_ctl -D "$PGDATA" restart -m fast -o "-p $PGPORT"
        ;;
    status)
        pg_ctl -D "$PGDATA" status
        ;;
    reload)
        pg_ctl -D "$PGDATA" reload
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|reload}"
        exit 1
esac
```

### Создание тестового кластера

```bash
#!/bin/bash

# Создание директории
mkdir -p /tmp/pgtest

# Инициализация кластера
pg_ctl -D /tmp/pgtest initdb -o "--encoding=UTF8"

# Настройка для тестирования
cat >> /tmp/pgtest/postgresql.conf << EOF
port = 5555
listen_addresses = 'localhost'
max_connections = 10
shared_buffers = 128MB
log_statement = 'all'
EOF

# Запуск
pg_ctl -D /tmp/pgtest start -l /tmp/pgtest/postgres.log

echo "Test cluster running on port 5555"
```

### Graceful restart с проверкой

```bash
#!/bin/bash

PGDATA="/var/lib/postgresql/15/main"

# Проверка синтаксиса конфигурации
if ! postgres -D "$PGDATA" -C config_file > /dev/null 2>&1; then
    echo "Configuration error detected!"
    exit 1
fi

# Graceful restart
pg_ctl -D "$PGDATA" restart -m fast -w -t 120

if [ $? -eq 0 ]; then
    echo "PostgreSQL restarted successfully"
else
    echo "Failed to restart PostgreSQL"
    exit 1
fi
```

## Работа с PID-файлом

pg_ctl использует файл `postmaster.pid` в PGDATA:

```bash
# Расположение PID-файла
cat /var/lib/postgresql/15/main/postmaster.pid
```

Содержимое файла:
```
1234                    # PID процесса
/var/lib/postgresql/15/main  # Data directory
1705312200              # Timestamp запуска
5432                    # Port
/var/run/postgresql     # Socket directory
localhost               # Listen address
  5432001     196608    # Shared memory info
ready                   # Status
```

### Очистка stale PID-файла

Если сервер не запускается из-за "stale" PID-файла:

```bash
# Проверить, действительно ли процесс не существует
ps aux | grep postgres

# Если процесс не найден — удалить PID-файл
rm /var/lib/postgresql/15/main/postmaster.pid

# Повторить запуск
pg_ctl -D /var/lib/postgresql/15/main start
```

## Best Practices

### 1. Всегда используйте -w для production

```bash
# Гарантирует, что сервер полностью запустился
pg_ctl -D "$PGDATA" start -w
```

### 2. Используйте fast shutdown по умолчанию

```bash
# Безопасно для production, не ждёт бесконечно
pg_ctl -D "$PGDATA" stop -m fast
```

### 3. Логируйте вывод в файл

```bash
# Для отладки проблем с запуском
pg_ctl -D "$PGDATA" -l "$PGDATA/startup.log" start
```

### 4. Устанавливайте разумный таймаут

```bash
# Для больших баз с длительным recovery
pg_ctl -D "$PGDATA" -t 300 -w start
```

### 5. Проверяйте статус после операций

```bash
pg_ctl -D "$PGDATA" start -w && pg_ctl -D "$PGDATA" status
```

## Типичные ошибки

### 1. Забыть указать PGDATA

**Ошибка:**
```
pg_ctl: no database directory specified and environment variable PGDATA unset
```

**Решение:**
```bash
export PGDATA=/var/lib/postgresql/15/main
# или
pg_ctl -D /var/lib/postgresql/15/main status
```

### 2. Запуск от неправильного пользователя

**Ошибка:**
```
pg_ctl: cannot be run as root
```

**Решение:**
```bash
sudo -u postgres pg_ctl -D /var/lib/postgresql/15/main start
```

### 3. Использование immediate shutdown без необходимости

```bash
# Неправильно — может привести к повреждению данных
pg_ctl -D "$PGDATA" stop -m immediate

# Правильно — для обычных ситуаций
pg_ctl -D "$PGDATA" stop -m fast
```

### 4. Неправильные права на директорию данных

```bash
# Проверка прав
ls -la /var/lib/postgresql/15/main

# Исправление
sudo chown -R postgres:postgres /var/lib/postgresql/15/main
sudo chmod 700 /var/lib/postgresql/15/main
```

### 5. Попытка запуска без инициализации

**Ошибка:**
```
pg_ctl: directory "/path/to/data" is not a database cluster directory
```

**Решение:**
```bash
pg_ctl -D /path/to/data initdb
```

## Сравнение с другими инструментами

| Возможность | pg_ctl | systemctl | pg_ctlcluster |
|-------------|--------|-----------|---------------|
| Кроссплатформенность | Да | Только Linux | Только Debian/Ubuntu |
| Интеграция с ОС | Нет | Да | Да |
| Автозапуск | Нет | Да | Да |
| Множественные кластеры | Вручную | Да | Да (встроенно) |
| Логирование | В файл | journald | В файл |

## Коды возврата

- `0` — операция успешна
- `1` — ошибка
- `2` — неверные параметры командной строки

```bash
pg_ctl -D "$PGDATA" status
if [ $? -eq 0 ]; then
    echo "Server is running"
else
    echo "Server is not running"
fi
```

## Заключение

pg_ctl — это фундаментальный инструмент для управления PostgreSQL, который:
- Работает на всех платформах
- Предоставляет низкоуровневый контроль над сервером
- Необходим для нестандартных установок
- Используется как основа для более высокоуровневых инструментов

Для production-систем на современных Linux рекомендуется использовать systemd, но знание pg_ctl необходимо для отладки, скриптов и работы на других платформах.
