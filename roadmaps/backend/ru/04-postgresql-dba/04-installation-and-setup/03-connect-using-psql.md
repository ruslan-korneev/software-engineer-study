# Подключение к PostgreSQL с помощью psql

[prev: 02-package-managers](./02-package-managers.md) | [next: 04-cloud-deployment](./04-cloud-deployment.md)
---

## Введение

**psql** — это официальный интерактивный терминальный клиент для PostgreSQL. Он позволяет выполнять SQL-запросы, управлять базами данных и администрировать сервер через командную строку.

## Установка psql

### Ubuntu/Debian

```bash
# Только клиент (без сервера)
sudo apt install postgresql-client-16

# Проверка
psql --version
```

### RHEL/CentOS

```bash
sudo dnf install postgresql16
```

### macOS

```bash
# Homebrew
brew install libpq
brew link --force libpq

# Или вместе с сервером
brew install postgresql@16
```

### Windows

psql устанавливается вместе с PostgreSQL или можно скачать отдельно:
- Установщик с [postgresql.org](https://www.postgresql.org/download/windows/)
- Добавьте `C:\Program Files\PostgreSQL\16\bin` в PATH

## Базовое подключение

### Синтаксис команды

```bash
psql [OPTIONS] [DBNAME [USERNAME]]
```

### Основные параметры подключения

| Параметр | Описание | Пример |
|----------|----------|--------|
| `-h, --host` | Хост сервера | `-h localhost` |
| `-p, --port` | Порт (по умолчанию 5432) | `-p 5432` |
| `-U, --username` | Имя пользователя | `-U postgres` |
| `-d, --dbname` | Имя базы данных | `-d mydb` |
| `-W, --password` | Запрос пароля | `-W` |
| `-w, --no-password` | Не запрашивать пароль | `-w` |

### Примеры подключения

```bash
# Локальное подключение (peer authentication)
sudo -u postgres psql

# Подключение к конкретной базе данных
psql -U postgres -d mydb

# Подключение к удалённому серверу
psql -h 192.168.1.100 -p 5432 -U myuser -d mydb

# Подключение через Unix-сокет
psql -h /var/run/postgresql -U postgres

# Подключение с запросом пароля
psql -h localhost -U myuser -d mydb -W
```

### Connection String (URI)

```bash
# Формат URI
psql "postgresql://user:password@host:port/database"

# Примеры
psql "postgresql://postgres:secret@localhost:5432/mydb"
psql "postgres://myuser:mypass@db.example.com/production"

# С дополнительными параметрами
psql "postgresql://user:pass@host/db?sslmode=require"
psql "postgresql://user@host/db?connect_timeout=10"
```

## Переменные окружения

Для удобства можно использовать переменные окружения:

```bash
# Установка переменных
export PGHOST=localhost
export PGPORT=5432
export PGUSER=myuser
export PGDATABASE=mydb
export PGPASSWORD=mypassword  # НЕ рекомендуется для production

# Теперь можно подключаться без параметров
psql

# Файл .pgpass (рекомендуется вместо PGPASSWORD)
# ~/.pgpass — формат: hostname:port:database:username:password
echo "localhost:5432:mydb:myuser:mypassword" >> ~/.pgpass
chmod 600 ~/.pgpass
```

### Файл .pgpass

```bash
# Создание файла
touch ~/.pgpass
chmod 600 ~/.pgpass

# Формат записей (каждая строка — отдельное подключение)
# hostname:port:database:username:password
localhost:5432:*:postgres:admin123
db.example.com:5432:production:appuser:secret
*:*:*:readonly:readonlypass
```

## Мета-команды psql

Мета-команды начинаются с обратного слэша (`\`) и не являются SQL.

### Информация о подключении

```
\conninfo       -- Информация о текущем подключении
\c dbname       -- Переключиться на другую базу данных
\c dbname user  -- Переключиться с другим пользователем
\c dbname user host port  -- Полное переключение
```

### Справка

```
\?              -- Справка по мета-командам
\h              -- Справка по SQL-командам
\h SELECT       -- Справка по конкретной команде
```

### Листинг объектов

```
\l              -- Список баз данных
\l+             -- Расширенный список баз данных

\dn             -- Список схем
\dn+            -- Расширенный список схем

\dt             -- Список таблиц в текущей схеме
\dt+            -- Расширенный список таблиц
\dt schema.*    -- Таблицы в конкретной схеме
\dt *.*         -- Все таблицы во всех схемах

\di             -- Список индексов
\ds             -- Список последовательностей (sequences)
\dv             -- Список представлений (views)
\dm             -- Список materialized views
\df             -- Список функций
\df+            -- Расширенная информация о функциях

\du             -- Список пользователей/ролей
\du+            -- Расширенная информация о ролях

\dp             -- Список привилегий на таблицы
\ddp            -- Привилегии по умолчанию
```

### Описание объектов

```
\d tablename       -- Структура таблицы
\d+ tablename      -- Расширенная информация о таблице
\d *               -- Все объекты
\d+ schemaname.*   -- Все объекты в схеме с подробностями
```

### Работа с запросами

```
\e              -- Открыть редактор для запроса
\e filename     -- Редактировать файл
\i filename     -- Выполнить SQL из файла
\ir filename    -- Выполнить файл (относительный путь)

\g              -- Выполнить последний запрос
\g filename     -- Выполнить и сохранить результат в файл

\p              -- Показать содержимое буфера запроса
\r              -- Очистить буфер запроса
\w filename     -- Записать буфер в файл
```

### Форматирование вывода

```
\x              -- Переключить расширенный вывод (вертикальный)
\x auto         -- Автоматический выбор формата
\x on           -- Всегда вертикальный вывод
\x off          -- Всегда горизонтальный вывод

\a              -- Переключить выровненный вывод
\t              -- Показывать только строки (без заголовков)
\f ','          -- Установить разделитель полей

\pset border 2  -- Стиль границ таблицы (0, 1, 2)
\pset format html  -- Вывод в HTML
\pset format csv   -- Вывод в CSV
\pset null 'NULL'  -- Отображение NULL значений
```

### Экспорт и импорт данных

```
\copy table TO 'file.csv' CSV HEADER;
\copy table FROM 'file.csv' CSV HEADER;

\copy (SELECT * FROM users WHERE active) TO 'active_users.csv' CSV HEADER;
```

### Переменные psql

```
\set            -- Показать все переменные
\set MYVAR value  -- Установить переменную
\echo :MYVAR    -- Вывести значение переменной

-- Использование в запросах
\set table_name 'users'
SELECT * FROM :table_name;

-- Полезные встроенные переменные
\set AUTOCOMMIT off  -- Отключить автокоммит
\set ON_ERROR_STOP on  -- Остановка при ошибке
\set VERBOSITY verbose  -- Подробные сообщения об ошибках
```

### Таймеры и статистика

```
\timing         -- Включить/выключить измерение времени запросов
\timing on      -- Включить
\timing off     -- Выключить
```

### Завершение работы

```
\q              -- Выход из psql
\quit           -- Выход из psql
```

## Полезные комбинации команд

### Администрирование

```sql
-- Просмотр активных подключений
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state != 'idle';

-- Размер баз данных
SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname))
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Размер таблиц
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 10;
```

### Работа со скриптами

```bash
# Выполнение SQL-файла
psql -U postgres -d mydb -f script.sql

# Выполнение одной команды
psql -U postgres -d mydb -c "SELECT version();"

# Несколько команд
psql -U postgres -d mydb -c "SELECT 1;" -c "SELECT 2;"

# Вывод без форматирования (для скриптов)
psql -U postgres -d mydb -t -A -c "SELECT count(*) FROM users;"

# Экспорт в CSV
psql -U postgres -d mydb -c "COPY (SELECT * FROM users) TO STDOUT CSV HEADER" > users.csv

# Параметры для скриптов
psql -U postgres -d mydb \
  --no-align \
  --tuples-only \
  --quiet \
  -c "SELECT username FROM users;"
```

### Параметры для автоматизации

| Параметр | Описание |
|----------|----------|
| `-t, --tuples-only` | Только данные (без заголовков) |
| `-A, --no-align` | Без выравнивания |
| `-q, --quiet` | Минимальный вывод |
| `-f file` | Выполнить файл |
| `-c command` | Выполнить команду |
| `-o file` | Перенаправить вывод в файл |
| `-1, --single-transaction` | Выполнить всё в одной транзакции |
| `-v var=value` | Установить переменную |

## Настройка .psqlrc

Файл `~/.psqlrc` позволяет настроить psql по умолчанию:

```sql
-- ~/.psqlrc

-- Отключить сообщение при запуске
\set QUIET 1

-- Настройка промпта
\set PROMPT1 '%[%033[1;32m%]%n@%/%R%# %[%033[0m%]'
\set PROMPT2 '%[%033[1;32m%]%R%# %[%033[0m%]'

-- Включить измерение времени
\timing on

-- Подробные сообщения об ошибках
\set VERBOSITY verbose

-- Остановка при ошибке в скриптах
\set ON_ERROR_STOP on

-- Отображение NULL
\pset null '[NULL]'

-- Автоматический расширенный вывод
\x auto

-- История команд
\set HISTFILE ~/.psql_history- :DBNAME
\set HISTCONTROL ignoredups
\set HISTSIZE 10000

-- Включить вывод обратно
\set QUIET 0

-- Приветствие
\echo 'Welcome to PostgreSQL!\n'
\echo 'Type :? for help.\n'

-- Полезные алиасы
\set version 'SELECT version();'
\set extensions 'SELECT * FROM pg_available_extensions;'
\set activity 'SELECT pid, usename, application_name, client_addr, state, query FROM pg_stat_activity;'
\set dbsize 'SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;'
```

Использование алиасов:

```
mydb=# :version
mydb=# :activity
mydb=# :dbsize
```

## Работа с SSL

```bash
# Подключение с SSL
psql "sslmode=require host=db.example.com dbname=mydb user=myuser"

# С клиентскими сертификатами
psql "sslmode=verify-full \
      sslrootcert=/path/to/ca.crt \
      sslcert=/path/to/client.crt \
      sslkey=/path/to/client.key \
      host=db.example.com \
      dbname=mydb \
      user=myuser"
```

### Режимы SSL

| Режим | Описание |
|-------|----------|
| `disable` | Без SSL |
| `allow` | SSL если сервер требует |
| `prefer` | SSL предпочтительно (по умолчанию) |
| `require` | SSL обязательно |
| `verify-ca` | SSL + проверка CA |
| `verify-full` | SSL + проверка CA + hostname |

## Устранение проблем

### Типичные ошибки

```bash
# Ошибка: connection refused
# Решение: проверьте, запущен ли PostgreSQL
sudo systemctl status postgresql

# Ошибка: peer authentication failed
# Решение: используйте -h localhost или измените pg_hba.conf
psql -h localhost -U postgres

# Ошибка: password authentication failed
# Решение: проверьте пароль или используйте .pgpass
psql -U postgres -W

# Ошибка: FATAL: database "xxx" does not exist
# Решение: укажите существующую базу данных
psql -U postgres -d postgres
```

### Диагностика подключения

```bash
# Проверка доступности порта
nc -zv localhost 5432
telnet localhost 5432

# Проверка конфигурации
sudo -u postgres psql -c "SHOW listen_addresses;"
sudo -u postgres psql -c "SHOW port;"

# Просмотр pg_hba.conf
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules;"
```

## Best Practices

1. **Используйте .pgpass** вместо PGPASSWORD для хранения паролей
2. **Настройте .psqlrc** для повышения продуктивности
3. **Используйте алиасы** для частых запросов
4. **Включите \timing** для мониторинга производительности
5. **Используйте \x auto** для автоматического форматирования
6. **Используйте SSL** при подключении к удалённым серверам
7. **Используйте -1** (single-transaction) для выполнения скриптов
8. **Сохраняйте историю** команд для каждой базы отдельно

## Полезные ссылки

- [Официальная документация psql](https://www.postgresql.org/docs/current/app-psql.html)
- [PostgreSQL Connection Strings](https://www.postgresql.org/docs/current/libpq-connect.html)

---
[prev: 02-package-managers](./02-package-managers.md) | [next: 04-cloud-deployment](./04-cloud-deployment.md)