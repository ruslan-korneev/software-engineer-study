# Безопасность Redis

## Содержание
1. [Модель безопасности Redis](#модель-безопасности-redis)
2. [Аутентификация](#аутентификация)
3. [Сетевая безопасность](#сетевая-безопасность)
4. [TLS/SSL шифрование](#tlsssl-шифрование)
5. [Опасные команды](#опасные-команды)
6. [Типичные уязвимости](#типичные-уязвимости)
7. [Checklist безопасности для production](#checklist-безопасности-для-production)
8. [Мониторинг и аудит](#мониторинг-и-аудит)

---

## Модель безопасности Redis

### Философия безопасности Redis

Redis изначально проектировался для работы в **доверенной среде** (trusted environment). Это означает:

- Redis предполагает, что доступ к нему имеют только доверенные клиенты
- Основная защита — сетевая изоляция, а не встроенные механизмы безопасности
- Redis НЕ должен быть напрямую доступен из интернета

### Уровни защиты

```
┌─────────────────────────────────────────────────────────────┐
│                    Внешний интернет                         │
├─────────────────────────────────────────────────────────────┤
│  Уровень 1: Firewall (iptables, security groups)            │
├─────────────────────────────────────────────────────────────┤
│  Уровень 2: Сетевая изоляция (VPC, private network)         │
├─────────────────────────────────────────────────────────────┤
│  Уровень 3: TLS шифрование                                  │
├─────────────────────────────────────────────────────────────┤
│  Уровень 4: Аутентификация (AUTH / ACL)                     │
├─────────────────────────────────────────────────────────────┤
│  Уровень 5: Авторизация (ACL permissions)                   │
├─────────────────────────────────────────────────────────────┤
│                       Redis Server                          │
└─────────────────────────────────────────────────────────────┘
```

### Принцип минимальных привилегий

Redis поддерживает принцип наименьших привилегий начиная с версии 6.0:

| Версия Redis | Возможности безопасности |
|--------------|-------------------------|
| < 6.0        | Только пароль (requirepass) |
| 6.0+         | ACL, пользователи, гранулярные права |
| 6.2+         | Улучшенные ACL, pub/sub ограничения |
| 7.0+         | ACL v2, селекторы, sharded pub/sub ACL |

---

## Аутентификация

### Классическая AUTH с паролем

Простейший метод аутентификации — единый пароль для всех подключений.

#### Настройка в redis.conf

```conf
# Установка пароля
requirepass your_super_strong_password_here_123!@#

# Рекомендуемая длина пароля: минимум 32 символа
# Используйте генератор: openssl rand -base64 32
```

#### Подключение с паролем

```bash
# Через redis-cli
redis-cli -a your_password

# Или после подключения
redis-cli
127.0.0.1:6379> AUTH your_password
OK

# Без аутентификации — ошибка
127.0.0.1:6379> GET key
(error) NOAUTH Authentication required.
```

#### Программное подключение (Python)

```python
import redis

# Вариант 1: пароль в конструкторе
r = redis.Redis(
    host='localhost',
    port=6379,
    password='your_password',
    decode_responses=True
)

# Вариант 2: через URL
r = redis.from_url('redis://:your_password@localhost:6379/0')

# Проверка подключения
r.ping()  # True
```

### ACL (Access Control Lists) — Redis 6+

ACL — это современная система управления доступом, позволяющая создавать пользователей с различными правами.

#### Основные концепции ACL

```
┌─────────────────────────────────────────────────────────┐
│                     ACL User                            │
├─────────────────────────────────────────────────────────┤
│  • Username (имя пользователя)                          │
│  • Password(s) (один или несколько паролей)             │
│  • Enabled/Disabled (активен/отключен)                  │
│  • Command permissions (разрешенные команды)            │
│  • Key patterns (доступные ключи)                       │
│  • Channel patterns (доступные каналы pub/sub)          │
└─────────────────────────────────────────────────────────┘
```

#### Просмотр текущих пользователей

```bash
# Список всех пользователей
127.0.0.1:6379> ACL LIST
1) "user default on nopass ~* &* +@all"

# Информация о текущем пользователе
127.0.0.1:6379> ACL WHOAMI
"default"

# Детальная информация о пользователе
127.0.0.1:6379> ACL GETUSER default
```

### Создание пользователей и назначение прав

#### Синтаксис ACL SETUSER

```bash
ACL SETUSER <username> [rule [rule ...]]
```

#### Основные правила (rules)

| Правило | Описание |
|---------|----------|
| `on` | Активировать пользователя |
| `off` | Деактивировать пользователя |
| `>password` | Добавить пароль |
| `<password` | Удалить пароль |
| `nopass` | Разрешить вход без пароля |
| `resetpass` | Сбросить все пароли |
| `~pattern` | Разрешить доступ к ключам по паттерну |
| `%R~pattern` | Только чтение ключей по паттерну |
| `%W~pattern` | Только запись ключей по паттерну |
| `allkeys` или `~*` | Доступ ко всем ключам |
| `resetkeys` | Сбросить все паттерны ключей |
| `&pattern` | Разрешить каналы pub/sub |
| `allchannels` | Все каналы |
| `+command` | Разрешить команду |
| `-command` | Запретить команду |
| `+@category` | Разрешить категорию команд |
| `-@category` | Запретить категорию команд |
| `allcommands` или `+@all` | Все команды |
| `nocommands` | Запретить все команды |

#### Примеры создания пользователей

```bash
# 1. Администратор с полными правами
ACL SETUSER admin on >admin_password ~* &* +@all

# 2. Пользователь только для чтения
ACL SETUSER readonly on >readonly_pass ~* -@all +@read +@connection

# 3. Пользователь для конкретного приложения (только определенные ключи)
ACL SETUSER app_user on >app_pass ~app:* ~cache:* +@all -@dangerous

# 4. Пользователь для сессий (только SET/GET/DEL для session:*)
ACL SETUSER session_user on >session_pass ~session:* +get +set +del +expire +ttl

# 5. Pub/Sub пользователь
ACL SETUSER pubsub_user on >pubsub_pass &notifications:* +subscribe +publish +psubscribe

# 6. Replica пользователь (для репликации)
ACL SETUSER replica on >replica_pass +psync +replconf +ping
```

#### Пользователь с несколькими паролями

```bash
# Добавление нескольких паролей (для ротации)
ACL SETUSER myuser on >password1 >password2 ~* +@all

# Удаление старого пароля
ACL SETUSER myuser <password1
```

#### Подключение под пользователем

```bash
# AUTH с именем пользователя (Redis 6+)
127.0.0.1:6379> AUTH username password
OK

# Или при подключении
redis-cli --user username --pass password
```

```python
# Python
r = redis.Redis(
    host='localhost',
    port=6379,
    username='app_user',
    password='app_pass'
)
```

### Категории команд

Redis группирует команды в категории для удобного управления правами.

#### Список категорий

```bash
127.0.0.1:6379> ACL CAT
 1) "keyspace"
 2) "read"
 3) "write"
 4) "set"
 5) "sortedset"
 6) "list"
 7) "hash"
 8) "string"
 9) "bitmap"
10) "hyperloglog"
11) "geo"
12) "stream"
13) "pubsub"
14) "admin"
15) "fast"
16) "slow"
17) "blocking"
18) "dangerous"
19) "connection"
20) "transaction"
21) "scripting"
```

#### Команды в категории

```bash
# Посмотреть команды в категории
127.0.0.1:6379> ACL CAT dangerous
1) "keys"
2) "flushdb"
3) "flushall"
4) "debug"
5) "shutdown"
6) "migrate"
...
```

#### Важные категории

| Категория | Описание | Примеры команд |
|-----------|----------|----------------|
| `@read` | Команды чтения | GET, HGET, LRANGE, SMEMBERS |
| `@write` | Команды записи | SET, HSET, LPUSH, SADD |
| `@admin` | Административные | CONFIG, SAVE, BGSAVE, DEBUG |
| `@dangerous` | Опасные команды | KEYS, FLUSHALL, SHUTDOWN |
| `@slow` | Медленные команды | KEYS, SMEMBERS на больших данных |
| `@fast` | Быстрые O(1) команды | GET, SET, HGET |
| `@scripting` | Скрипты Lua | EVAL, EVALSHA, SCRIPT |
| `@pubsub` | Pub/Sub команды | PUBLISH, SUBSCRIBE |
| `@transaction` | Транзакции | MULTI, EXEC, WATCH |
| `@connection` | Подключение | AUTH, PING, SELECT, QUIT |

### Сохранение ACL

#### В файле redis.conf

```conf
# Прямое определение пользователей
user default on nopass ~* &* +@all
user admin on >strong_password ~* &* +@all
user readonly on >read_pass ~* +@read +@connection
```

#### В отдельном файле ACL

```conf
# redis.conf
aclfile /etc/redis/users.acl
```

```
# /etc/redis/users.acl
user default on nopass ~* &* +@all
user admin on #<sha256_hash_of_password> ~* &* +@all
user app on >app_password ~app:* +@all -@dangerous
```

#### Управление ACL файлом

```bash
# Сохранить текущие ACL в файл
127.0.0.1:6379> ACL SAVE
OK

# Загрузить ACL из файла
127.0.0.1:6379> ACL LOAD
OK
```

---

## Сетевая безопасность

### Bind к localhost

По умолчанию Redis слушает только localhost, что является первой линией защиты.

#### Настройка bind

```conf
# Только localhost (по умолчанию, безопасно)
bind 127.0.0.1 -::1

# Конкретный внутренний IP
bind 192.168.1.100

# Несколько интерфейсов
bind 127.0.0.1 192.168.1.100

# ВСЕ интерфейсы (ОПАСНО! Никогда не используйте в production без firewall)
bind 0.0.0.0

# Отключить bind (слушать все) — эквивалентно bind 0.0.0.0
# bind 127.0.0.1  # закомментировано
```

#### Рекомендации

```
┌─────────────────────────────────────────────────────────────┐
│  БЕЗОПАСНАЯ КОНФИГУРАЦИЯ                                    │
│                                                             │
│  • bind 127.0.0.1 — для локальной разработки               │
│  • bind <private_ip> — для внутренней сети                  │
│  • Никогда bind 0.0.0.0 без firewall и TLS                 │
└─────────────────────────────────────────────────────────────┘
```

### Protected Mode

Protected mode — дополнительный уровень защиты, включенный по умолчанию.

#### Как работает Protected Mode

```
┌─────────────────────────────────────────────────────────────┐
│               Protected Mode = YES                          │
├─────────────────────────────────────────────────────────────┤
│  Если:                                                      │
│    • НЕТ явного bind (слушает все интерфейсы)              │
│    • НЕТ пароля (requirepass)                               │
│  То:                                                        │
│    • Redis отвечает ТОЛЬКО на localhost подключения         │
│    • Внешние подключения получают ошибку                    │
└─────────────────────────────────────────────────────────────┘
```

#### Конфигурация

```conf
# Включить protected mode (по умолчанию)
protected-mode yes

# Отключить (требуется явный bind или пароль)
protected-mode no
```

#### Ошибка при срабатывании Protected Mode

```
(error) DENIED Redis is running in protected mode because protected
mode is enabled, no bind address was specified, no authentication
password is requested to clients. In this mode connections are only
accepted from the loopback interface.
```

### Firewall правила

#### iptables (Linux)

```bash
# Разрешить Redis только из внутренней сети
sudo iptables -A INPUT -p tcp --dport 6379 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -s 172.16.0.0/12 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -s 192.168.0.0/16 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP

# Разрешить только конкретные IP
sudo iptables -A INPUT -p tcp --dport 6379 -s 192.168.1.10 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -s 192.168.1.20 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP
```

#### UFW (Ubuntu)

```bash
# Разрешить из подсети
sudo ufw allow from 192.168.1.0/24 to any port 6379

# Разрешить конкретный IP
sudo ufw allow from 192.168.1.10 to any port 6379

# Запретить всем остальным
sudo ufw deny 6379
```

#### firewalld (CentOS/RHEL)

```bash
# Создать зону для Redis
sudo firewall-cmd --permanent --new-zone=redis
sudo firewall-cmd --permanent --zone=redis --add-source=192.168.1.0/24
sudo firewall-cmd --permanent --zone=redis --add-port=6379/tcp
sudo firewall-cmd --reload
```

#### AWS Security Groups

```json
{
  "SecurityGroupIngress": [
    {
      "IpProtocol": "tcp",
      "FromPort": 6379,
      "ToPort": 6379,
      "SourceSecurityGroupId": "sg-application-servers"
    }
  ]
}
```

---

## TLS/SSL шифрование

### Зачем нужен TLS

```
БЕЗ TLS:                              С TLS:
┌─────────┐   plaintext   ┌─────────┐  ┌─────────┐  encrypted  ┌─────────┐
│ Client  │──────────────>│  Redis  │  │ Client  │────────────>│  Redis  │
└─────────┘               └─────────┘  └─────────┘             └─────────┘
     │                                      │
     └── Пароли и данные                    └── Данные зашифрованы
         передаются открыто                     AES-256-GCM
```

### Настройка TLS

#### Генерация сертификатов

```bash
# 1. Создать CA (Certificate Authority)
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha256 -key ca.key -days 3650 \
    -out ca.crt -subj "/CN=Redis-CA"

# 2. Создать ключ и CSR для сервера
openssl genrsa -out redis.key 4096
openssl req -new -sha256 -key redis.key \
    -out redis.csr -subj "/CN=redis-server"

# 3. Подписать сертификат сервера
openssl x509 -req -sha256 -in redis.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -days 365 -out redis.crt

# 4. Создать DH параметры (опционально, для безопасности)
openssl dhparam -out redis.dh 2048

# 5. Установить права
chmod 600 redis.key ca.key
chmod 644 redis.crt ca.crt redis.dh
```

#### Конфигурация redis.conf

```conf
# Включить TLS на порту 6379
port 0
tls-port 6379

# Сертификаты
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt

# DH параметры (опционально)
tls-dh-params-file /etc/redis/tls/redis.dh

# Требовать клиентский сертификат
tls-auth-clients optional  # no | optional | yes

# Минимальная версия TLS
tls-protocols "TLSv1.2 TLSv1.3"

# Предпочитаемые шифры
tls-ciphers DEFAULT:!MEDIUM

# Предпочитать серверные шифры
tls-prefer-server-ciphers yes

# TLS для репликации
tls-replication yes

# TLS для кластера
tls-cluster yes
```

#### Подключение с TLS

```bash
# redis-cli с TLS
redis-cli --tls \
    --cert /path/to/client.crt \
    --key /path/to/client.key \
    --cacert /path/to/ca.crt \
    -p 6379

# Без клиентского сертификата (если tls-auth-clients = no)
redis-cli --tls --cacert /path/to/ca.crt
```

```python
# Python с TLS
import redis
import ssl

# Вариант 1: с клиентским сертификатом
r = redis.Redis(
    host='redis.example.com',
    port=6379,
    ssl=True,
    ssl_certfile='/path/to/client.crt',
    ssl_keyfile='/path/to/client.key',
    ssl_ca_certs='/path/to/ca.crt',
    ssl_cert_reqs=ssl.CERT_REQUIRED
)

# Вариант 2: только проверка сервера
r = redis.Redis(
    host='redis.example.com',
    port=6379,
    ssl=True,
    ssl_ca_certs='/path/to/ca.crt',
    ssl_cert_reqs=ssl.CERT_REQUIRED
)

# Вариант 3: через URL
r = redis.from_url(
    'rediss://user:password@redis.example.com:6379/0',  # rediss = redis + ssl
    ssl_ca_certs='/path/to/ca.crt'
)
```

### Сертификаты

#### Структура PKI для Redis

```
┌─────────────────────────────────────────────────────────────┐
│                    Root CA (ca.crt)                         │
│                         │                                   │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│   │ Redis    │   │ Redis    │   │ Client   │               │
│   │ Master   │   │ Replica  │   │ Certs    │               │
│   │ Cert     │   │ Cert     │   │          │               │
│   └──────────┘   └──────────┘   └──────────┘               │
└─────────────────────────────────────────────────────────────┘
```

#### Проверка сертификата

```bash
# Информация о сертификате
openssl x509 -in redis.crt -text -noout

# Проверка соответствия ключа и сертификата
openssl x509 -noout -modulus -in redis.crt | openssl md5
openssl rsa -noout -modulus -in redis.key | openssl md5
# MD5 хеши должны совпадать

# Тест TLS подключения
openssl s_client -connect localhost:6379 -CAfile ca.crt
```

---

## Опасные команды

### Список опасных команд

| Команда | Опасность | Описание |
|---------|-----------|----------|
| `FLUSHALL` | Критическая | Удаляет ВСЕ данные во ВСЕХ базах |
| `FLUSHDB` | Высокая | Удаляет все данные в текущей базе |
| `KEYS *` | Высокая | Блокирует Redis при большом количестве ключей |
| `DEBUG SEGFAULT` | Критическая | Аварийно завершает Redis |
| `DEBUG SLEEP` | Высокая | Блокирует сервер |
| `SHUTDOWN` | Критическая | Выключает Redis |
| `CONFIG SET` | Высокая | Изменяет конфигурацию на лету |
| `CONFIG REWRITE` | Высокая | Перезаписывает redis.conf |
| `SAVE` | Высокая | Блокирующее сохранение |
| `BGSAVE` | Средняя | Fork может потребовать много памяти |
| `MIGRATE` | Высокая | Может отправить данные на внешний сервер |
| `SLAVEOF` / `REPLICAOF` | Критическая | Превращает сервер в реплику |
| `DEBUG SET-ACTIVE-EXPIRE` | Средняя | Влияет на производительность |
| `SCRIPT KILL` | Средняя | Прерывает выполнение Lua скриптов |
| `MODULE LOAD` | Критическая | Загружает произвольный код |

### Переименование команд (rename-command)

#### Синтаксис

```conf
# redis.conf
rename-command ОПАСНАЯ_КОМАНДА НОВОЕ_ИМЯ
```

#### Примеры

```conf
# Переименовать в сложное имя
rename-command FLUSHALL "HIDDEN_FLUSHALL_e2d5h7k9"
rename-command FLUSHDB "HIDDEN_FLUSHDB_m3n8p4q2"
rename-command DEBUG "HIDDEN_DEBUG_a1b2c3d4"
rename-command SHUTDOWN "HIDDEN_SHUTDOWN_x9y8z7"
rename-command CONFIG "HIDDEN_CONFIG_f5g6h7i8"

# Использование после переименования
127.0.0.1:6379> HIDDEN_FLUSHALL_e2d5h7k9
OK
```

### Отключение команд

```conf
# Полностью отключить команду (переименовать в пустую строку)
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command KEYS ""
rename-command CONFIG ""
```

```bash
# Попытка использовать отключенную команду
127.0.0.1:6379> FLUSHALL
(error) ERR unknown command 'FLUSHALL', with args beginning with:
```

#### Использование ACL вместо rename-command (Redis 6+)

```bash
# Более гибкий подход — ACL
ACL SETUSER admin on >password ~* +@all

ACL SETUSER developer on >dev_pass ~* +@all -flushall -flushdb -debug -shutdown -keys

ACL SETUSER readonly on >read_pass ~* +@read +@connection
```

#### Рекомендации

```
┌─────────────────────────────────────────────────────────────┐
│  РЕКОМЕНДУЕМЫЕ ОТКЛЮЧЕНИЯ ДЛЯ PRODUCTION                    │
├─────────────────────────────────────────────────────────────┤
│  rename-command DEBUG ""                                    │
│  rename-command KEYS ""           # используйте SCAN        │
│  rename-command FLUSHALL ""       # или сложное имя         │
│  rename-command FLUSHDB ""        # или сложное имя         │
│  rename-command CONFIG ""         # или ограничьте ACL      │
│  rename-command SHUTDOWN ""       # управляйте через OS     │
└─────────────────────────────────────────────────────────────┘
```

---

## Типичные уязвимости

### Открытый Redis в интернете

#### Проблема

```
┌─────────────────────────────────────────────────────────────┐
│  КРИТИЧЕСКАЯ УЯЗВИМОСТЬ                                     │
├─────────────────────────────────────────────────────────────┤
│  bind 0.0.0.0                                               │
│  protected-mode no                                          │
│  requirepass <пусто>                                        │
│                                                             │
│  = ЛЮБОЙ человек в интернете может подключиться            │
└─────────────────────────────────────────────────────────────┘
```

#### Последствия

1. **Кража данных** — злоумышленник читает все ключи
2. **Удаление данных** — FLUSHALL уничтожает все
3. **Криптомайнинг** — загрузка модуля или использование ресурсов
4. **SSH ключи** — запись SSH public key для доступа к серверу
5. **Выполнение кода** — через Lua скрипты или модули

#### Атака с записью SSH ключа

```bash
# Атакующий подключается к открытому Redis
redis-cli -h victim.com

# Записывает свой SSH ключ
CONFIG SET dir /root/.ssh/
CONFIG SET dbfilename authorized_keys
SET ssh_key "\n\nssh-rsa AAAA...attacker_key...== attacker@evil\n\n"
SAVE

# Теперь атакующий может войти по SSH
ssh root@victim.com
```

#### Защита

```conf
# ОБЯЗАТЕЛЬНО:
bind 127.0.0.1
protected-mode yes
requirepass very_strong_password_32_chars_min

# ДОПОЛНИТЕЛЬНО:
rename-command CONFIG ""
rename-command DEBUG ""
```

### Слабые пароли

#### Примеры слабых паролей

```
❌ requirepass redis
❌ requirepass password
❌ requirepass 123456
❌ requirepass admin
❌ requirepass test
```

#### Генерация сильного пароля

```bash
# OpenSSL (рекомендуется)
openssl rand -base64 32
# Пример: Kx9mL3nP7qR2sT5uW8yZ1aB4cD6eF0gH

# /dev/urandom
head -c 32 /dev/urandom | base64
# Пример: 2Fg8Jk0lMn4pQr6sTu8vWx0yZ2Ab4Cd=

# pwgen
pwgen -s 32 1
# Пример: 7xK9mL3nP5qR8sT0uW2yZ4aB6cD1eF3g
```

#### Брутфорс защита

Redis НЕ имеет встроенной защиты от брутфорса! Используйте:
- Fail2ban
- Rate limiting на firewall
- Очень длинные пароли (32+ символов)

```ini
# /etc/fail2ban/jail.d/redis.conf
[redis]
enabled = true
port = 6379
filter = redis-auth
logpath = /var/log/redis/redis-server.log
maxretry = 3
bantime = 3600
```

### Эксплойты через EVAL

#### Lua песочница

Redis использует песочницу для Lua, но она не идеальна:

```lua
-- Доступные функции ограничены
redis.call('GET', 'key')  -- ОК
os.execute('rm -rf /')    -- ЗАБЛОКИРОВАНО
io.open('/etc/passwd')    -- ЗАБЛОКИРОВАНО
```

#### Потенциальные проблемы

```lua
-- 1. Бесконечный цикл (блокирует Redis)
EVAL "while true do end" 0

-- 2. Исчерпание памяти
EVAL "local t = {} for i=1,1000000000 do t[i] = string.rep('x', 1000) end" 0

-- 3. CPU-интенсивные операции
EVAL "local s = '' for i=1,100000 do s = s .. i end" 0
```

#### Защита от злоупотреблений EVAL

```conf
# Лимит времени выполнения Lua (в мс)
lua-time-limit 5000

# Отключить скрипты для непривилегированных пользователей (ACL)
ACL SETUSER app on >pass ~* +@all -@scripting
```

```bash
# Принудительное завершение скрипта
127.0.0.1:6379> SCRIPT KILL
```

---

## Checklist безопасности для production

### Минимальный Checklist

```
┌─────────────────────────────────────────────────────────────┐
│  МИНИМАЛЬНЫЕ ТРЕБОВАНИЯ                                     │
├─────────────────────────────────────────────────────────────┤
│  [ ] bind 127.0.0.1 или внутренний IP                       │
│  [ ] requirepass установлен (32+ символов)                  │
│  [ ] protected-mode yes                                     │
│  [ ] Firewall настроен (только нужные IP)                   │
│  [ ] Redis не запущен от root                               │
└─────────────────────────────────────────────────────────────┘
```

### Расширенный Checklist

#### Сеть и доступ

- [ ] Redis недоступен из интернета
- [ ] Настроен bind на конкретный интерфейс
- [ ] Protected mode включен
- [ ] Firewall ограничивает доступ
- [ ] VPC/Private network для cloud

#### Аутентификация

- [ ] Установлен сильный пароль (32+ символов)
- [ ] Используется ACL (Redis 6+)
- [ ] Отдельные пользователи для приложений
- [ ] Принцип минимальных привилегий
- [ ] Пароли не хранятся в коде (используйте env/secrets)

#### Шифрование

- [ ] TLS включен для production
- [ ] Сертификаты от доверенного CA
- [ ] TLS 1.2+ (отключены старые версии)
- [ ] Регулярная ротация сертификатов

#### Команды

- [ ] Опасные команды отключены/переименованы
- [ ] KEYS заменен на SCAN
- [ ] DEBUG отключен
- [ ] CONFIG ограничен

#### Операционная безопасность

- [ ] Redis запущен от непривилегированного пользователя
- [ ] Директории с правильными правами (700)
- [ ] RDB/AOF файлы защищены (600)
- [ ] Логи настроены и мониторятся
- [ ] Регулярные бэкапы

#### Мониторинг

- [ ] Алерты на подозрительные команды
- [ ] Мониторинг подключений
- [ ] Аудит команд (если требуется)

### Пример безопасной конфигурации

```conf
# =============================================================================
# PRODUCTION REDIS CONFIGURATION (security-focused)
# =============================================================================

# --- СЕТЬ ---
bind 10.0.1.100                    # Только внутренний IP
port 0                             # Отключить обычный порт
tls-port 6379                      # Только TLS
protected-mode yes
tcp-backlog 511
timeout 300                        # Закрывать idle соединения

# --- TLS ---
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients optional
tls-protocols "TLSv1.2 TLSv1.3"
tls-prefer-server-ciphers yes
tls-replication yes

# --- АУТЕНТИФИКАЦИЯ ---
aclfile /etc/redis/users.acl
# requirepass используется только для default user в ACL файле

# --- ОПАСНЫЕ КОМАНДЫ ---
rename-command DEBUG ""
rename-command SHUTDOWN ""
rename-command FLUSHALL "REALLY_FLUSHALL_a1b2c3d4"
rename-command FLUSHDB "REALLY_FLUSHDB_e5f6g7h8"
rename-command CONFIG "REDIS_CONFIG_i9j0k1l2"
rename-command KEYS ""

# --- ЛИМИТЫ ---
maxclients 10000
maxmemory 4gb
maxmemory-policy allkeys-lru
lua-time-limit 5000

# --- ЛОГИРОВАНИЕ ---
loglevel notice
logfile /var/log/redis/redis-server.log
```

```
# /etc/redis/users.acl
user default off
user admin on #<sha256hash> ~* &* +@all
user app_read on #<sha256hash> ~app:* +@read +@connection
user app_write on #<sha256hash> ~app:* +@all -@admin -@dangerous
user monitoring on #<sha256hash> ~* +info +ping +slowlog +client|list
```

---

## Мониторинг и аудит

### Мониторинг подключений

```bash
# Список клиентов
127.0.0.1:6379> CLIENT LIST
id=5 addr=127.0.0.1:52345 fd=8 name= age=1234 idle=0 flags=N db=0 ...

# Количество подключений
127.0.0.1:6379> INFO clients
# Clients
connected_clients:10
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0
```

### Мониторинг команд

```bash
# Реальтайм мониторинг всех команд
127.0.0.1:6379> MONITOR
OK
1679123456.123456 [0 127.0.0.1:52345] "GET" "user:1"
1679123456.234567 [0 127.0.0.1:52345] "SET" "user:1" "data"

# ВНИМАНИЕ: MONITOR влияет на производительность!
```

### Slowlog

```bash
# Получить медленные команды
127.0.0.1:6379> SLOWLOG GET 10
1) 1) (integer) 1                    # ID записи
   2) (integer) 1679123456           # Unix timestamp
   3) (integer) 15234                # Время выполнения (мкс)
   4) 1) "KEYS"                      # Команда
      2) "*"
   5) "127.0.0.1:52345"              # Адрес клиента
   6) "app_user"                     # Имя пользователя (Redis 6+)

# Настройка slowlog
CONFIG SET slowlog-log-slower-than 10000  # 10 мс
CONFIG SET slowlog-max-len 1000
```

### ACL Log (Redis 6+)

```bash
# Просмотр попыток нарушения ACL
127.0.0.1:6379> ACL LOG 10
1) 1) "count"
   2) (integer) 1
   3) "reason"
   4) "auth"                         # Неудачная аутентификация
   5) "context"
   6) "toplevel"
   7) "object"
   8) "AUTH"
   9) "username"
  10) "wrong_user"
  11) "age-seconds"
  12) "10.5"
  13) "client-info"
  14) "id=5 addr=192.168.1.100:54321 ..."

# Очистить ACL log
127.0.0.1:6379> ACL LOG RESET
OK
```

### Метрики безопасности

```bash
# Статистика
127.0.0.1:6379> INFO stats
total_connections_received:1234
rejected_connections:5               # Отклоненные подключения
total_commands_processed:567890

# Keyspace
127.0.0.1:6379> INFO keyspace
db0:keys=12345,expires=1000,avg_ttl=3600000
```

### Интеграция с системами мониторинга

#### Prometheus + redis_exporter

```yaml
# docker-compose.yml
services:
  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      REDIS_ADDR: "redis://redis:6379"
      REDIS_PASSWORD: "${REDIS_PASSWORD}"
    ports:
      - "9121:9121"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

#### Grafana алерты

```yaml
# Пример алерта на подозрительные команды
- alert: RedisFlushAllDetected
  expr: increase(redis_commands_total{cmd="flushall"}[5m]) > 0
  for: 0s
  labels:
    severity: critical
  annotations:
    summary: "FLUSHALL detected on {{ $labels.instance }}"
    description: "Someone executed FLUSHALL on Redis"
```

### Аудит через логи

#### Включение расширенного логирования

```conf
# redis.conf
loglevel verbose  # debug | verbose | notice | warning

# Логировать в файл
logfile /var/log/redis/redis-server.log
```

#### Централизованный сбор логов

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/redis/redis-server.log
  tags: ["redis"]

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "redis-logs-%{+yyyy.MM.dd}"
```

### Таблица важных метрик безопасности

| Метрика | Описание | Порог алерта |
|---------|----------|--------------|
| `rejected_connections` | Отклоненные подключения | > 10/мин |
| `connected_clients` | Текущие клиенты | > 80% от maxclients |
| `acl_log` (cmd denied) | Запрещенные команды | Любое появление |
| `acl_log` (auth failed) | Неудачные входы | > 5/мин |
| `slowlog` (KEYS, SMEMBERS) | Тяжелые операции | Появление в production |
| `used_memory` | Использование памяти | > 90% maxmemory |

---

## Заключение

Безопасность Redis строится на нескольких уровнях защиты:

1. **Сетевая изоляция** — Redis должен быть недоступен извне
2. **Аутентификация** — сильные пароли и ACL (Redis 6+)
3. **Шифрование** — TLS для защиты данных в транзите
4. **Минимальные привилегии** — только необходимые команды и ключи
5. **Мониторинг** — отслеживание подозрительной активности

Помните: Redis изначально проектировался для доверенной среды. Безопасность — это ваша ответственность!
