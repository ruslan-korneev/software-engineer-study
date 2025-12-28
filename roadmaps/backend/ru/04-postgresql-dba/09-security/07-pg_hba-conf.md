# Конфигурация pg_hba.conf

[prev: 06-roles](./06-roles.md) | [next: 08-ssl-settings](./08-ssl-settings.md)

---

## Введение

`pg_hba.conf` (PostgreSQL Host-Based Authentication) - это файл конфигурации, определяющий правила аутентификации клиентов при подключении к PostgreSQL. Этот файл контролирует, какие пользователи могут подключаться, с каких хостов и какой метод аутентификации будет использоваться.

## Расположение файла

```bash
# Типичные расположения
/var/lib/postgresql/15/main/pg_hba.conf      # Debian/Ubuntu
/var/lib/pgsql/15/data/pg_hba.conf           # RHEL/CentOS
/usr/local/pgsql/data/pg_hba.conf            # Сборка из исходников

# Найти через SQL
SHOW hba_file;

# Или через data_directory
SHOW data_directory;
```

## Формат записей

### Общий синтаксис

```
# TYPE     DATABASE     USER         ADDRESS            METHOD  [OPTIONS]
local      database     user                            method  [options]
host       database     user         address            method  [options]
hostssl    database     user         address            method  [options]
hostnossl  database     user         address            method  [options]
hostgssenc database     user         address            method  [options]
hostnogssenc database   user         address            method  [options]
```

### Типы подключений (TYPE)

| Тип | Описание |
|-----|----------|
| `local` | Unix domain socket (локальное подключение) |
| `host` | TCP/IP (любое: SSL и без SSL) |
| `hostssl` | Только TCP/IP с SSL |
| `hostnossl` | Только TCP/IP без SSL |
| `hostgssenc` | TCP/IP с GSSAPI шифрованием |
| `hostnogssenc` | TCP/IP без GSSAPI шифрования |

### Поле DATABASE

| Значение | Описание |
|----------|----------|
| `all` | Все базы данных |
| `sameuser` | БД с именем, совпадающим с именем пользователя |
| `samerole` | Пользователь - член роли с именем БД |
| `replication` | Репликационные подключения |
| `database_name` | Конкретная база данных |
| `db1,db2,db3` | Список баз данных |
| `@filename` | Файл со списком баз данных |

### Поле USER

| Значение | Описание |
|----------|----------|
| `all` | Все пользователи |
| `username` | Конкретный пользователь |
| `+rolename` | Любой член роли |
| `user1,user2` | Список пользователей |
| `@filename` | Файл со списком пользователей |

### Поле ADDRESS

| Формат | Пример | Описание |
|--------|--------|----------|
| IP/mask | `192.168.1.0/24` | CIDR нотация |
| IP mask | `192.168.1.0 255.255.255.0` | Классическая маска |
| hostname | `client.example.com` | Имя хоста (обратный DNS) |
| `.domain` | `.example.com` | Все хосты в домене |
| `samehost` | - | IP адреса этого сервера |
| `samenet` | - | Подсети этого сервера |
| `all` | - | Любой адрес |

### Методы аутентификации

| Метод | Описание |
|-------|----------|
| `trust` | Без проверки (доверие) |
| `reject` | Отклонить подключение |
| `scram-sha-256` | SCRAM аутентификация (рекомендуется) |
| `md5` | MD5 хэш пароля |
| `password` | Пароль открытым текстом |
| `peer` | UID Unix сокета |
| `ident` | Ident протокол |
| `gss` | GSSAPI/Kerberos |
| `sspi` | Windows SSPI |
| `ldap` | LDAP сервер |
| `radius` | RADIUS сервер |
| `cert` | SSL сертификат клиента |
| `pam` | PAM модули |

## Примеры конфигураций

### Базовая безопасная конфигурация

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Локальные подключения через Unix socket
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 локальные подключения
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 локальные подключения
host    all             all             ::1/128                 scram-sha-256

# Репликация (только с определённых серверов)
hostssl replication     repl_user       10.0.0.0/8              scram-sha-256

# Отклонить всё остальное
host    all             all             0.0.0.0/0               reject
```

### Продуктовая конфигурация

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# === ЛОКАЛЬНЫЕ ПОДКЛЮЧЕНИЯ ===
# Суперпользователь только через peer
local   all             postgres                                peer

# Остальные локальные через пароль
local   all             all                                     scram-sha-256

# === РЕПЛИКАЦИЯ ===
# Только SSL, только с известных реплик
hostssl replication     repl_user       10.0.1.10/32            scram-sha-256
hostssl replication     repl_user       10.0.1.11/32            scram-sha-256

# === ВНУТРЕННЯЯ СЕТЬ ===
# Приложения (только SSL)
hostssl app_db          app_user        10.0.0.0/16             scram-sha-256

# Мониторинг
hostssl all             monitoring      10.0.0.0/16             scram-sha-256

# DBA с доступом по сертификату
hostssl all             dba             10.0.0.0/16             cert

# === ВНЕШНИЕ ПОДКЛЮЧЕНИЯ ===
# VPN клиенты
hostssl all             all             172.16.0.0/12           scram-sha-256

# === БЛОКИРОВКА ===
# Всё остальное отклонить
host    all             all             0.0.0.0/0               reject
hostssl all             all             0.0.0.0/0               reject
```

### Конфигурация с LDAP

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Локальные подключения
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# Корпоративные пользователи через LDAP
hostssl all             all             192.168.0.0/16          ldap ldapserver=ldap.company.com ldapbasedn="dc=company,dc=com" ldapsearchattribute=uid ldapbinddn="cn=pg_service,dc=company,dc=com" ldapbindpasswd=secret

# Сервисные аккаунты через пароль
hostssl all             +service_accounts 10.0.0.0/8            scram-sha-256
```

## Порядок обработки правил

**ВАЖНО:** PostgreSQL обрабатывает правила сверху вниз и применяет **первое совпадающее** правило!

```
# Порядок важен!

# Правильный порядок:
# 1. Специфичные правила (конкретный пользователь/БД/адрес)
# 2. Более общие правила
# 3. Правило-заглушка (reject all)

# Пример ПРАВИЛЬНОГО порядка:
hostssl  myapp       admin           10.0.0.5/32     cert      # Специфичный
hostssl  myapp       all             10.0.0.0/24     scram-sha-256  # Средний
hostssl  all         all             10.0.0.0/8      scram-sha-256  # Общий
host     all         all             0.0.0.0/0       reject    # Заглушка

# Пример НЕПРАВИЛЬНОГО порядка:
host     all         all             0.0.0.0/0       reject    # Всё заблокировано!
hostssl  myapp       admin           10.0.0.5/32     cert      # Никогда не сработает
```

## Опции методов аутентификации

### LDAP опции

```
hostssl all all 0.0.0.0/0 ldap
  ldapserver=ldap.example.com
  ldapport=389
  ldapscheme=ldaps
  ldaptls=1
  ldapbasedn="ou=users,dc=example,dc=com"
  ldapsearchattribute=uid
  ldapbinddn="cn=service,dc=example,dc=com"
  ldapbindpasswd=secret
```

### RADIUS опции

```
host all all 0.0.0.0/0 radius
  radiusservers="radius1.example.com,radius2.example.com"
  radiussecrets="secret1,secret2"
  radiusports=1812,1812
  radiusidentifiers="pg-server"
```

### GSSAPI опции

```
hostgssenc all all 0.0.0.0/0 gss
  include_realm=0
  krb_realm=COMPANY.COM
  map=gss_map
```

### Cert опции

```
hostssl all all 0.0.0.0/0 cert
  clientcert=verify-full
  map=cert_map
```

### PAM опции

```
host all all 0.0.0.0/0 pam
  pamservice=postgresql
```

## pg_ident.conf - маппинг пользователей

Файл `pg_ident.conf` используется для сопоставления внешних имён пользователей (из ОС, сертификатов, LDAP) с именами ролей PostgreSQL.

### Формат

```
# MAPNAME       SYSTEM-USERNAME         PG-USERNAME
```

### Примеры

```
# Простой маппинг
mymap           john                    app_user
mymap           jane                    app_user

# Регулярные выражения
mymap           /^(.*)@company\.com$    \1
mymap           /^admin_(.*)$           \1

# Для сертификатов
cert_map        /^CN=(.*)$              \1
cert_map        service_cert            service_user

# Для peer/ident
osmap           root                    postgres
osmap           /^(.*)$                 \1
```

### Использование в pg_hba.conf

```
local   all     all                     peer map=osmap
hostssl all     all     0.0.0.0/0       cert map=cert_map
```

## Применение изменений

### Перезагрузка конфигурации

```sql
-- Через SQL
SELECT pg_reload_conf();

-- Или через psql
\! pg_ctl reload

-- Или через systemd
-- sudo systemctl reload postgresql
```

### Проверка ошибок

```sql
-- Просмотр текущих правил
SELECT * FROM pg_hba_file_rules;

-- Поля:
-- line_number, type, database, user_name, address, netmask, auth_method, options, error

-- Проверка на ошибки
SELECT line_number, error
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

### Проверка без применения

```bash
# PostgreSQL 16+
pg_hba_validate /path/to/pg_hba.conf

# Или запустить сервер в режиме проверки
postgres --check -D /path/to/data
```

## Отладка подключений

### Логирование

```sql
-- postgresql.conf
log_connections = on
log_disconnections = on
log_hostname = on

-- Для детальной отладки
log_min_messages = debug5
```

### Проверка правила

```sql
-- Показать какое правило применяется
-- (только после успешного подключения)
SELECT
    client_addr,
    client_port,
    backend_type,
    usename,
    datname,
    ssl,
    ssl_version,
    ssl_cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE pid = pg_backend_pid();
```

### Тестирование подключения

```bash
# Простая проверка
psql -h hostname -U username -d database -c "SELECT 1"

# С отладкой SSL
PGSSLMODE=require psql -h hostname -U username -d database

# Verbose режим
psql -h hostname -U username -d database --echo-all
```

## Best Practices

### 1. Всегда заканчивайте reject правилом

```
# Последнее правило
host    all     all     0.0.0.0/0       reject
```

### 2. Используйте hostssl вместо host

```
# Плохо
host    all     all     0.0.0.0/0       scram-sha-256

# Хорошо
hostssl all     all     0.0.0.0/0       scram-sha-256
```

### 3. Ограничивайте доступ суперпользователя

```
# Только локально
local   all     postgres                peer

# Или с определённого IP администратора
hostssl all     postgres    10.0.0.5/32 scram-sha-256
```

### 4. Используйте scram-sha-256 вместо md5

```
# Устаревший
host    all     all     10.0.0.0/8      md5

# Современный
host    all     all     10.0.0.0/8      scram-sha-256
```

### 5. Группируйте правила логически

```
# === ЛОКАЛЬНЫЕ ПОДКЛЮЧЕНИЯ ===
local   all     postgres                peer
local   all     all                     scram-sha-256

# === РЕПЛИКАЦИЯ ===
hostssl replication repl    10.0.1.0/24 scram-sha-256

# === ПРИЛОЖЕНИЯ ===
hostssl app_db  app_user    10.0.0.0/16 scram-sha-256

# === АДМИНИСТРИРОВАНИЕ ===
hostssl all     dba         10.0.0.0/16 cert

# === БЛОКИРОВКА ===
host    all     all         0.0.0.0/0   reject
```

### 6. Документируйте изменения

```
# 2024-01-15: Добавлен доступ для нового сервера приложений
# Задача: JIRA-1234
# Ответственный: admin@company.com
hostssl app_db  app_user    10.0.2.50/32    scram-sha-256
```

### 7. Используйте файлы для списков

```
# pg_hba.conf
hostssl all     @/etc/postgresql/allowed_users.txt    10.0.0.0/8    scram-sha-256

# /etc/postgresql/allowed_users.txt
alice
bob
charlie
```

## Типичные ошибки

### 1. Неправильный порядок правил

```
# Ошибка: общее правило перед специфичным
host    all     all         0.0.0.0/0       reject
hostssl mydb    myuser      10.0.0.1/32     scram-sha-256  # Никогда не сработает!

# Правильно
hostssl mydb    myuser      10.0.0.1/32     scram-sha-256
host    all     all         0.0.0.0/0       reject
```

### 2. Забыли перезагрузить конфигурацию

```sql
-- После изменения pg_hba.conf ОБЯЗАТЕЛЬНО:
SELECT pg_reload_conf();
```

### 3. Опечатка в IP адресе

```
# Ошибка в маске
host    all     all     192.168.1.0/24      scram-sha-256
# vs
host    all     all     192.168.1.0/16      scram-sha-256  # Совсем другой диапазон!
```

### 4. Путаница между local и host

```
# local - Unix socket (без ADDRESS)
local   all     all                         peer

# host - TCP/IP (нужен ADDRESS)
host    all     all     127.0.0.1/32        scram-sha-256

# Ошибка:
host    all     all                         peer  # Синтаксическая ошибка!
```

### 5. Отсутствие правила для IPv6

```
# Забыли про IPv6
host    all     all     127.0.0.1/32        scram-sha-256

# Нужно также:
host    all     all     ::1/128             scram-sha-256
```

## Миграция и обновление

### Переход с md5 на scram-sha-256

```sql
-- 1. Изменить password_encryption
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- 2. Обновить пароли пользователей
ALTER USER app_user WITH PASSWORD 'same_password';

-- 3. Изменить pg_hba.conf
-- Заменить md5 на scram-sha-256

-- 4. Перезагрузить конфигурацию
SELECT pg_reload_conf();
```

### Проверка совместимости клиентов

```bash
# Старые клиенты могут не поддерживать SCRAM
# Проверить версию libpq
pg_config --version

# SCRAM поддерживается с PostgreSQL 10+
```

## Связанные темы

- [Модели аутентификации](./05-authentication-models.md) - детали методов аутентификации
- [SSL Settings](./08-ssl-settings.md) - настройка SSL соединений
- [Роли](./06-roles.md) - управление пользователями

---

[prev: 06-roles](./06-roles.md) | [next: 08-ssl-settings](./08-ssl-settings.md)
