# PgBouncer

## Введение

**PgBouncer** — это легковесный пулер соединений для PostgreSQL. Он располагается между клиентами и сервером PostgreSQL, управляя пулом соединений и значительно снижая накладные расходы на установку новых подключений к базе данных.

### Зачем нужен Connection Pooling?

Каждое соединение с PostgreSQL потребляет ресурсы:
- **Память**: каждый backend-процесс PostgreSQL занимает около 5-10 МБ RAM
- **CPU**: форк нового процесса требует процессорного времени
- **Время**: установка соединения занимает 100-200 мс (с SSL — ещё больше)

При большом количестве короткоживущих соединений (типично для веб-приложений) эти накладные расходы становятся критичными.

```
Без пулера:                    С PgBouncer:

Client 1 ──► PostgreSQL        Client 1 ──┐
Client 2 ──► PostgreSQL        Client 2 ──┼──► PgBouncer ──► PostgreSQL
Client 3 ──► PostgreSQL        Client 3 ──┘    (10 conn)    (3 conn)
   ...                            ...
Client N ──► PostgreSQL        Client N ──┘
(N connections)                (N clients, M connections, M << N)
```

## Установка PgBouncer

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install pgbouncer
```

### CentOS/RHEL

```bash
sudo yum install pgbouncer
# или
sudo dnf install pgbouncer
```

### macOS

```bash
brew install pgbouncer
```

### Docker

```bash
docker run -d \
  --name pgbouncer \
  -e DATABASE_URL="postgres://user:pass@postgres:5432/dbname" \
  -p 6432:6432 \
  edoburu/pgbouncer
```

## Конфигурация

Основной конфигурационный файл: `/etc/pgbouncer/pgbouncer.ini`

### Базовая конфигурация

```ini
[databases]
; Формат: dbname = connection_string
mydb = host=localhost port=5432 dbname=mydb

; Можно задавать алиасы
production = host=prod-server port=5432 dbname=app_production
staging = host=staging-server port=5432 dbname=app_staging

; Wildcard — все базы данных
* = host=localhost port=5432

[pgbouncer]
; Адрес и порт для прослушивания
listen_addr = 0.0.0.0
listen_port = 6432

; Аутентификация
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Режим пулинга
pool_mode = transaction

; Ограничения пула
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

; Логирование
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

; Административный доступ
admin_users = postgres, admin
stats_users = stats, monitoring
```

### Файл аутентификации (userlist.txt)

```
; Формат: "username" "password"
"myuser" "md5hash_or_plaintext"
"admin" "scram-sha-256$iterations:salt$StoredKey:ServerKey"
```

Для генерации MD5 хеша:

```bash
# Формат MD5: md5 + md5(password + username)
echo -n "passwordusername" | md5sum
# Добавить префикс "md5"
```

Автоматическая генерация из PostgreSQL:

```sql
SELECT usename, passwd FROM pg_shadow;
```

## Режимы пулинга (Pool Modes)

### 1. Session Mode (session)

```ini
pool_mode = session
```

Соединение с сервером закрепляется за клиентом на всё время сессии.

**Плюсы:**
- Полная совместимость с PostgreSQL
- Поддержка prepared statements, temporary tables, session variables

**Минусы:**
- Минимальная экономия соединений
- Соединение занято, даже когда клиент idle

**Когда использовать:**
- Приложения с долгими сессиями
- Необходимость использования SET, prepared statements

### 2. Transaction Mode (transaction)

```ini
pool_mode = transaction
```

Соединение закрепляется за клиентом только на время транзакции.

**Плюсы:**
- Максимальная эффективность использования соединений
- Один серверный коннект может обслуживать много клиентов

**Минусы:**
- Нельзя использовать:
  - Prepared statements (без server_reset_query)
  - SET commands
  - LISTEN/NOTIFY
  - Advisory locks
  - Temporary tables

**Когда использовать:**
- Веб-приложения с короткими транзакциями
- Высоконагруженные системы

### 3. Statement Mode (statement)

```ini
pool_mode = statement
```

Соединение возвращается в пул после каждого запроса.

**Плюсы:**
- Максимальная утилизация соединений

**Минусы:**
- Нельзя использовать транзакции (каждый statement — отдельная транзакция)
- Очень ограниченная совместимость

**Когда использовать:**
- Только для read-only нагрузки
- Простые запросы без транзакций

### Сравнение режимов

| Функция | Session | Transaction | Statement |
|---------|---------|-------------|-----------|
| Prepared statements | ✅ | ❌ | ❌ |
| SET commands | ✅ | ❌ | ❌ |
| Transactions | ✅ | ✅ | ❌ |
| LISTEN/NOTIFY | ✅ | ❌ | ❌ |
| Advisory locks | ✅ | ❌ | ❌ |
| Temp tables | ✅ | ❌ | ❌ |
| Эффективность пула | Низкая | Высокая | Максимальная |

## Параметры пула

### Основные параметры

```ini
[pgbouncer]
; Максимальное число клиентских соединений
max_client_conn = 1000

; Размер пула на каждую пару user/database
default_pool_size = 20

; Минимальное число соединений в пуле (всегда готовы)
min_pool_size = 5

; Резервный пул для пиковых нагрузок
reserve_pool_size = 5
reserve_pool_timeout = 3

; Максимальное число соединений на одну БД
max_db_connections = 50

; Максимальное число соединений от одного пользователя
max_user_connections = 50
```

### Таймауты

```ini
; Таймаут на получение соединения из пула
query_timeout = 120

; Таймаут ожидания свободного соединения
client_idle_timeout = 0

; Закрыть соединение если клиент idle
client_login_timeout = 60

; Время жизни серверного соединения
server_lifetime = 3600

; Таймаут на idle серверное соединение
server_idle_timeout = 600

; Таймаут на подключение к серверу
server_connect_timeout = 15

; Таймаут на логин к серверу
server_login_retry = 15
```

### Очистка состояния

```ini
; SQL выполняемый после каждой транзакции (в transaction mode)
server_reset_query = DISCARD ALL

; Проверка работоспособности соединения
server_check_query = SELECT 1

; Интервал проверки
server_check_delay = 30
```

## Мониторинг и администрирование

### Подключение к административной консоли

```bash
psql -h localhost -p 6432 -U admin pgbouncer
```

### Полезные команды

```sql
-- Показать статистику по базам данных
SHOW DATABASES;

-- Показать статистику по пулам
SHOW POOLS;

-- Показать активные клиенты
SHOW CLIENTS;

-- Показать серверные соединения
SHOW SERVERS;

-- Показать текущую конфигурацию
SHOW CONFIG;

-- Показать статистику
SHOW STATS;

-- Показать средние значения статистики
SHOW STATS_AVERAGES;

-- Перезагрузить конфигурацию
RELOAD;

-- Приостановить базу данных
PAUSE mydb;

-- Возобновить
RESUME mydb;

-- Отключить базу данных
DISABLE mydb;

-- Включить
ENABLE mydb;

-- Завершить работу (после обработки текущих запросов)
SHUTDOWN;
```

### Пример вывода SHOW POOLS

```
  database  |   user    | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait |  pool_mode
------------+-----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+-------------
 mydb       | myuser    |        15 |          0 |        10 |       5 |       0 |         0 |        0 |       0 | transaction
```

- `cl_active` — активные клиенты
- `cl_waiting` — клиенты, ожидающие соединение
- `sv_active` — активные серверные соединения
- `sv_idle` — простаивающие серверные соединения
- `maxwait` — максимальное время ожидания (секунды)

## Prometheus и мониторинг

### Экспортер метрик

Используйте [pgbouncer_exporter](https://github.com/prometheus-community/pgbouncer_exporter):

```bash
pgbouncer_exporter --pgBouncer.connectionString="postgres://admin@localhost:6432/pgbouncer?sslmode=disable"
```

### Ключевые метрики для мониторинга

```yaml
# Prometheus alerts
groups:
  - name: pgbouncer
    rules:
      - alert: PgBouncerClientsWaiting
        expr: pgbouncer_pools_client_waiting_connections > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Клиенты ожидают соединения"

      - alert: PgBouncerMaxConnections
        expr: pgbouncer_pools_server_active_connections / pgbouncer_config_max_client_conn > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Достигнут лимит соединений"
```

## Best Practices

### 1. Правильный размер пула

```ini
; Формула: pool_size = (количество ядер CPU PostgreSQL * 2) + число дисков
; Для 8 ядер и SSD: 8 * 2 + 1 = 17, округляем до 20
default_pool_size = 20

; Оставить запас для PostgreSQL
; max_connections (PostgreSQL) = default_pool_size * databases * users + reserve
```

### 2. Настройка для веб-приложений

```ini
[pgbouncer]
pool_mode = transaction
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

; Быстрое обнаружение разрыва соединений
server_check_delay = 10
tcp_keepalive = 1
tcp_keepidle = 30
tcp_keepintvl = 10

; Очистка состояния
server_reset_query = DISCARD ALL
```

### 3. Высокая доступность

```ini
; Несколько серверов с приоритетом
[databases]
mydb = host=primary,replica1,replica2 port=5432 dbname=mydb

; Или использовать DNS round-robin
mydb = host=pgcluster.example.com port=5432 dbname=mydb
```

### 4. Безопасность

```ini
[pgbouncer]
; Использовать SSL
client_tls_sslmode = require
client_tls_key_file = /etc/pgbouncer/server.key
client_tls_cert_file = /etc/pgbouncer/server.crt

; SSL к PostgreSQL
server_tls_sslmode = verify-full
server_tls_ca_file = /etc/pgbouncer/ca.crt

; Ограничить административный доступ
admin_users = admin
stats_users = monitoring
auth_type = scram-sha-256
```

## Типичные ошибки

### 1. Prepared statements в transaction mode

**Проблема:** Ошибка "prepared statement does not exist"

**Решение:**
```ini
; Использовать server_reset_query для очистки
server_reset_query = DISCARD ALL

; Или переключить на session mode для данного приложения
[databases]
legacy_app = host=localhost dbname=mydb pool_mode=session
```

### 2. Слишком большой пул

**Проблема:** PostgreSQL перегружен из-за большого числа соединений

**Решение:**
```ini
; Ограничить общее число соединений
max_db_connections = 100

; Проверить max_connections в PostgreSQL
; postgresql.conf: max_connections = 150
```

### 3. Клиенты в очереди (cl_waiting > 0)

**Проблема:** Клиенты ожидают свободное соединение

**Решение:**
```ini
; Увеличить размер пула
default_pool_size = 30

; Использовать резервный пул
reserve_pool_size = 10
reserve_pool_timeout = 3

; Оптимизировать запросы (уменьшить время транзакций)
```

### 4. Потеря соединений

**Проблема:** Соединения неожиданно закрываются

**Решение:**
```ini
; Включить TCP keepalive
tcp_keepalive = 1
tcp_keepidle = 30
tcp_keepintvl = 10
tcp_keepcnt = 3

; Увеличить server_lifetime
server_lifetime = 3600
```

## Пример production-конфигурации

```ini
[databases]
; Production база
app = host=pg-primary.internal port=5432 dbname=app_production
app_ro = host=pg-replica.internal port=5432 dbname=app_production

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432

; Аутентификация
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

; Пулинг
pool_mode = transaction
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 3

; Лимиты
max_client_conn = 2000
max_db_connections = 100
max_user_connections = 100

; Таймауты
query_timeout = 300
query_wait_timeout = 60
client_idle_timeout = 600
server_lifetime = 3600
server_idle_timeout = 300

; Очистка
server_reset_query = DISCARD ALL
server_check_query = SELECT 1
server_check_delay = 30

; TCP
tcp_keepalive = 1
tcp_keepidle = 30
tcp_keepintvl = 10
tcp_keepcnt = 3

; SSL (клиенты)
client_tls_sslmode = require
client_tls_key_file = /etc/pgbouncer/tls/server.key
client_tls_cert_file = /etc/pgbouncer/tls/server.crt

; SSL (PostgreSQL)
server_tls_sslmode = verify-full
server_tls_ca_file = /etc/pgbouncer/tls/ca.crt

; Логирование
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

; Администрирование
admin_users = pgbouncer_admin
stats_users = monitoring
ignore_startup_parameters = extra_float_digits
```

## Интеграция с приложениями

### Django

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'pgbouncer-host',
        'PORT': '6432',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'CONN_MAX_AGE': 0,  # Важно! Не кешировать соединения
        'OPTIONS': {
            'options': '-c statement_timeout=30000',
        },
    }
}
```

### SQLAlchemy

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@pgbouncer:6432/mydb",
    pool_size=0,  # Отключить пул SQLAlchemy
    pool_pre_ping=True,
    connect_args={
        "options": "-c statement_timeout=30000"
    }
)
```

### Node.js (pg)

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'pgbouncer-host',
  port: 6432,
  database: 'mydb',
  user: 'myuser',
  password: 'mypassword',
  max: 1,  // Минимальный пул на стороне клиента
  idleTimeoutMillis: 0,
  connectionTimeoutMillis: 5000,
});
```

## Заключение

PgBouncer — это мощный и надёжный инструмент для управления соединениями PostgreSQL. Правильная настройка позволяет:

- Обслуживать тысячи клиентов с минимальным числом реальных соединений к БД
- Снизить нагрузку на PostgreSQL
- Ускорить время отклика за счёт переиспользования соединений
- Повысить стабильность под высокой нагрузкой

Ключ к успешному использованию — правильный выбор режима пулинга и настройка параметров под конкретную нагрузку вашего приложения.
