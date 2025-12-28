# Анализ логов PostgreSQL

[prev: 02-profiling-tools](./02-profiling-tools.md) | [next: 04-system-views](./04-system-views.md)

---

Логи PostgreSQL содержат важную информацию о работе базы данных: ошибки, медленные запросы, подключения и многое другое. Правильная настройка и анализ логов критически важны для диагностики проблем.

## Настройка логирования PostgreSQL

### Основные параметры (postgresql.conf)

```ini
#------------------------------------------------------------------------------
# РАСПОЛОЖЕНИЕ И ФОРМАТ ЛОГОВ
#------------------------------------------------------------------------------

# Куда писать логи: stderr, csvlog, jsonlog, syslog, eventlog
log_destination = 'stderr'

# Включить сборщик логов (logging collector)
logging_collector = on

# Директория для лог-файлов
log_directory = 'log'

# Шаблон имени файла
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

# Права на файлы логов
log_file_mode = 0600

# Ротация логов
log_rotation_age = 1d          # Новый файл каждый день
log_rotation_size = 100MB      # Или при достижении 100MB
log_truncate_on_rotation = off # Не перезаписывать старые

#------------------------------------------------------------------------------
# ЧТО ЛОГИРОВАТЬ
#------------------------------------------------------------------------------

# Уровень детализации: debug5-debug1, info, notice, warning, error, log, fatal, panic
log_min_messages = warning

# Уровень для клиентских сообщений
client_min_messages = notice

# Логировать запросы с ошибками
log_min_error_statement = error

# Логировать медленные запросы (мс, 0 = отключено)
log_min_duration_statement = 1000  # Запросы > 1 секунды

# Альтернатива: логировать N% запросов
log_statement_sample_rate = 0.01  # 1% всех запросов
log_min_duration_sample = 100     # Из них только > 100мс

#------------------------------------------------------------------------------
# ДЕТАЛИЗАЦИЯ ЛОГОВ
#------------------------------------------------------------------------------

# Какие запросы логировать: none, ddl, mod, all
log_statement = 'none'

# Логировать продолжительность запросов
log_duration = off

# Логировать подключения и отключения
log_connections = on
log_disconnections = on

# Логировать имя хоста клиента
log_hostname = off

# Логировать использование временных файлов > N кБ
log_temp_files = 0  # Все временные файлы

# Логировать контрольные точки
log_checkpoints = on

# Логировать блокировки дольше N мс
log_lock_waits = on
deadlock_timeout = 1s

# Логировать автовакуум
log_autovacuum_min_duration = 0  # Все операции автовакуума
```

### Формат строки лога

```ini
# Префикс каждой строки лога
log_line_prefix = '%t [%p-%l] %q%u@%d '

# Расшифровка escape-последовательностей:
# %t - timestamp
# %p - PID процесса
# %l - номер строки в сессии
# %q - пустая строка для backend, space для других
# %u - имя пользователя
# %d - имя базы данных
# %a - имя приложения
# %h - хост клиента
# %r - хост:порт клиента
# %c - ID сессии
# %e - SQL state код
# %i - тип команды
```

**Рекомендуемый формат для анализа:**

```ini
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

### CSV формат логов

```ini
# Включение CSV логирования
log_destination = 'csvlog'
logging_collector = on

# CSV файлы можно загружать в таблицу для анализа
```

```sql
-- Создание таблицы для импорта CSV логов
CREATE TABLE postgres_log (
    log_time timestamp(3) with time zone,
    user_name text,
    database_name text,
    process_id integer,
    connection_from text,
    session_id text,
    session_line_num bigint,
    command_tag text,
    session_start_time timestamp with time zone,
    virtual_transaction_id text,
    transaction_id bigint,
    error_severity text,
    sql_state_code text,
    message text,
    detail text,
    hint text,
    internal_query text,
    internal_query_pos integer,
    context text,
    query text,
    query_pos integer,
    location text,
    application_name text,
    backend_type text
);

-- Загрузка CSV лога
COPY postgres_log FROM '/var/log/postgresql/postgresql.csv' WITH CSV;
```

## pgBadger - анализатор логов

pgBadger - мощный инструмент для анализа логов PostgreSQL, создающий подробные HTML-отчёты.

### Установка

```bash
# Debian/Ubuntu
sudo apt install pgbadger

# CentOS/RHEL
sudo yum install pgbadger

# Или через CPAN
cpan App::pgBadger

# Или из исходников
git clone https://github.com/darold/pgbadger.git
cd pgbadger
perl Makefile.PL
make && sudo make install
```

### Базовое использование

```bash
# Анализ одного лог-файла
pgbadger /var/log/postgresql/postgresql-2024-01-15.log -o report.html

# Анализ нескольких файлов
pgbadger /var/log/postgresql/postgresql-*.log -o report.html

# Анализ сжатых логов
pgbadger /var/log/postgresql/postgresql-*.log.gz -o report.html

# С указанием формата префикса
pgbadger --prefix '%t [%p]: [%l-1] user=%u,db=%d ' \
    postgresql.log -o report.html
```

### Расширенные параметры

```bash
# Инкрементальный анализ (для больших логов)
pgbadger -I -O /var/reports/pgbadger/ postgresql.log

# Анализ за определённый период
pgbadger --begin "2024-01-15 10:00:00" --end "2024-01-15 18:00:00" \
    postgresql.log -o report.html

# Только определённая база данных
pgbadger --dbname mydb postgresql.log -o report.html

# Только определённый пользователь
pgbadger --dbuser admin postgresql.log -o report.html

# Исключить запросы определённого типа
pgbadger --exclude-query "^COPY" postgresql.log -o report.html

# JSON формат для дальнейшей обработки
pgbadger -f json postgresql.log -o report.json

# Топ N медленных запросов
pgbadger --top 50 postgresql.log -o report.html
```

### Автоматический отчёт по расписанию

```bash
#!/bin/bash
# /etc/cron.daily/pgbadger

LOGDIR=/var/log/postgresql
OUTDIR=/var/www/html/pgbadger
DATE=$(date -d "yesterday" +%Y-%m-%d)

pgbadger ${LOGDIR}/postgresql-${DATE}*.log \
    -o ${OUTDIR}/report-${DATE}.html \
    --title "PostgreSQL Report ${DATE}"

# Создание индексного файла
pgbadger -I -O ${OUTDIR}/ ${LOGDIR}/postgresql-*.log
```

### Содержимое отчёта pgBadger

Отчёт включает:
- **Overview** - общая статистика
- **Connections** - подключения по времени
- **Sessions** - статистика сессий
- **Checkpoints** - информация о контрольных точках
- **Temporary files** - использование временных файлов
- **Vacuuming** - операции очистки
- **Locks** - статистика блокировок
- **Queries** - анализ запросов
  - Самые медленные запросы
  - Самые частые запросы
  - Запросы с наибольшим временем

## Ручной анализ логов

### Поиск ошибок

```bash
# Найти все ошибки
grep -i "error\|fatal\|panic" postgresql.log

# Ошибки с контекстом
grep -B 2 -A 5 "ERROR:" postgresql.log

# Подсчёт ошибок по типу
grep "ERROR:" postgresql.log | sed 's/.*ERROR:  //' | \
    cut -d':' -f1 | sort | uniq -c | sort -rn

# Ошибки за последний час
grep "$(date -d '1 hour ago' +'%Y-%m-%d %H')" postgresql.log | grep ERROR
```

### Анализ медленных запросов

```bash
# Найти запросы > 1 секунды (log_min_duration_statement = 1000)
grep "duration:" postgresql.log | awk '$4 > 1000'

# Топ-10 самых медленных запросов
grep "duration:" postgresql.log | \
    sed 's/.*duration: \([0-9.]*\).*/\1/' | \
    sort -rn | head -10

# Распределение запросов по времени выполнения
grep "duration:" postgresql.log | \
    awk '{
        d = $NF;
        if (d < 100) bucket="<100ms";
        else if (d < 1000) bucket="100ms-1s";
        else if (d < 5000) bucket="1s-5s";
        else bucket=">5s";
        print bucket
    }' | sort | uniq -c
```

### Анализ подключений

```bash
# Количество подключений за день
grep "connection authorized" postgresql.log | wc -l

# Подключения по пользователям
grep "connection authorized" postgresql.log | \
    grep -oP 'user=\K\w+' | sort | uniq -c | sort -rn

# Неудачные попытки подключения
grep "authentication failed\|connection refused" postgresql.log

# Распределение подключений по времени
grep "connection authorized" postgresql.log | \
    cut -d' ' -f1,2 | cut -d':' -f1,2 | uniq -c
```

### Анализ блокировок

```bash
# Найти deadlock
grep -i "deadlock detected" postgresql.log

# Ожидания блокировок
grep "still waiting for" postgresql.log

# Блокировки с контекстом
grep -B 5 -A 10 "deadlock detected" postgresql.log
```

## Типичные ошибки и их анализ

### Ошибки подключения

```
FATAL:  too many connections for role "app_user"
# Решение: увеличить connection limit для роли или max_connections

FATAL:  remaining connection slots are reserved for non-replication superuser connections
# Решение: увеличить max_connections или superuser_reserved_connections

FATAL:  password authentication failed for user "admin"
# Решение: проверить пароль и настройки pg_hba.conf
```

### Ошибки памяти

```
ERROR:  out of memory
DETAIL:  Failed on request of size 123456.
# Решение: увеличить work_mem или оптимизировать запрос

ERROR:  could not resize shared memory segment
# Решение: проверить параметры shared_buffers и системные лимиты
```

### Ошибки блокировок

```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on transaction 67890; blocked by process 11111.
# Решение: анализ транзакций и порядка блокировок

LOG:  process 12345 still waiting for AccessShareLock on relation 16384 after 1000.123 ms
# Решение: найти блокирующую транзакцию через pg_stat_activity
```

### Ошибки дискового пространства

```
PANIC:  could not write to file "pg_wal/xlogtemp.12345": No space left on device
# Решение: срочно освободить место, проверить pg_wal

ERROR:  could not extend file "base/16384/12345": No space left on device
# Решение: освободить место, проверить автовакуум
```

## Скрипт мониторинга логов

```bash
#!/bin/bash
# log_monitor.sh - мониторинг ошибок в реальном времени

LOGFILE=/var/log/postgresql/postgresql.log
ALERT_EMAIL="dba@example.com"

tail -F "$LOGFILE" | while read line; do
    # Проверка на критические ошибки
    if echo "$line" | grep -qE "FATAL|PANIC"; then
        echo "$line" | mail -s "PostgreSQL CRITICAL ERROR" "$ALERT_EMAIL"
    fi

    # Проверка на deadlock
    if echo "$line" | grep -qi "deadlock"; then
        echo "$line" | mail -s "PostgreSQL Deadlock Detected" "$ALERT_EMAIL"
    fi

    # Проверка на OOM
    if echo "$line" | grep -qi "out of memory"; then
        echo "$line" | mail -s "PostgreSQL Out of Memory" "$ALERT_EMAIL"
    fi
done
```

## Лучшие практики логирования

1. **Включите CSV формат** для удобного анализа в базе данных
2. **Настройте log_min_duration_statement** для отслеживания медленных запросов
3. **Включите log_checkpoints** для мониторинга контрольных точек
4. **Включите log_lock_waits** для отслеживания блокировок
5. **Используйте pgBadger** для регулярного анализа
6. **Ротируйте логи** чтобы не заполнить диск
7. **Мониторьте логи в реальном времени** для критических систем
8. **Храните логи достаточно долго** для анализа трендов

## Заключение

Правильно настроенное логирование и регулярный анализ логов позволяют:
- Быстро находить и диагностировать проблемы
- Отслеживать производительность запросов
- Выявлять паттерны ошибок
- Планировать оптимизации на основе данных

Рекомендуется автоматизировать анализ логов с помощью pgBadger и настроить алерты для критических ошибок.

---

[prev: 02-profiling-tools](./02-profiling-tools.md) | [next: 04-system-views](./04-system-views.md)
