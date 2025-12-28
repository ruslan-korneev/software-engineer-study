# Модели аутентификации в PostgreSQL

[prev: 04-row-level-security](./04-row-level-security.md) | [next: 06-roles](./06-roles.md)

---

## Введение

Аутентификация - это процесс проверки подлинности пользователя, пытающегося подключиться к базе данных. PostgreSQL поддерживает множество методов аутентификации, от простых (trust, password) до корпоративных (LDAP, Kerberos, RADIUS). Выбор метода зависит от требований безопасности и инфраструктуры организации.

## Обзор методов аутентификации

| Метод | Описание | Безопасность | Применение |
|-------|----------|--------------|------------|
| `trust` | Без проверки | Минимальная | Разработка, локальные подключения |
| `reject` | Отклоняет все подключения | - | Блокировка доступа |
| `password` | Пароль открытым текстом | Низкая | Только с SSL |
| `md5` | MD5 хэш пароля | Средняя | Устаревший |
| `scram-sha-256` | Современный протокол | Высокая | Рекомендуется |
| `peer` | UID Unix сокета | Высокая | Локальные подключения |
| `ident` | Ident протокол | Средняя | Unix системы |
| `gss` | Kerberos/GSSAPI | Высокая | Корпоративные среды |
| `sspi` | Windows SSPI | Высокая | Windows AD |
| `ldap` | LDAP сервер | Высокая | Централизованная аутентификация |
| `radius` | RADIUS сервер | Высокая | Сетевая аутентификация |
| `cert` | SSL сертификаты | Высокая | Машинная аутентификация |
| `pam` | PAM модули | Зависит от модуля | Гибкая интеграция |

## Trust аутентификация

Самый простой метод - доверяет любому подключению без проверки.

```
# pg_hba.conf
# ВНИМАНИЕ: Использовать только для разработки!
local   all   all                 trust
host    all   all   127.0.0.1/32  trust
```

**Когда использовать:**
- Локальная разработка
- Изолированные тестовые среды
- Временное восстановление доступа

**Риски:**
- Любой пользователь системы может подключиться как любой пользователь PostgreSQL

## Password методы

### password (cleartext)

Пароль передаётся открытым текстом. **Использовать только с SSL!**

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  password
```

### md5

Хэширует пароль с использованием MD5. Устаревший, но всё ещё распространённый.

```
# pg_hba.conf
host  all  all  192.168.1.0/24  md5
```

Как работает:
1. Клиент получает случайную соль от сервера
2. Вычисляет: `md5(md5(password + username) + salt)`
3. Отправляет хэш серверу

**Недостатки:**
- MD5 считается криптографически слабым
- Уязвим к replay-атакам при перехвате

### scram-sha-256 (рекомендуется)

Современный метод на основе SCRAM (Salted Challenge Response Authentication Mechanism).

```
# pg_hba.conf
host  all  all  0.0.0.0/0  scram-sha-256
```

```sql
-- Настройка для хранения паролей в SCRAM формате
SET password_encryption = 'scram-sha-256';

-- Создание пользователя
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Проверка формата хранения
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'app_user';
-- rolpassword начинается с "SCRAM-SHA-256$..."
```

**Преимущества:**
- Использует SHA-256 вместо MD5
- Защита от replay-атак
- Сервер не хранит пароль в открытом виде
- Взаимная аутентификация клиента и сервера

**Настройка перехода с md5 на scram-sha-256:**

```sql
-- 1. Изменить настройку шифрования
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- 2. Обновить пароли пользователей
ALTER USER app_user WITH PASSWORD 'same_or_new_password';

-- 3. Обновить pg_hba.conf
-- Заменить md5 на scram-sha-256
```

## Peer аутентификация

Проверяет, что имя пользователя операционной системы совпадает с именем пользователя PostgreSQL.

```
# pg_hba.conf - только для локальных подключений
local  all  all  peer
```

Работает через Unix domain socket, получая UID подключающегося процесса.

```bash
# Если пользователь ОС - postgres, он может подключиться как postgres
$ whoami
postgres
$ psql -U postgres
# Подключение успешно

# Но не может подключиться как другой пользователь
$ psql -U admin
# ERROR: Peer authentication failed
```

**Маппинг пользователей:**

```
# pg_hba.conf
local  all  all  peer map=my_map

# pg_ident.conf - маппинг пользователей ОС на PostgreSQL
# MAPNAME    SYSTEM-USERNAME    PG-USERNAME
my_map       john               app_user
my_map       /^(.*)@domain$     \1
```

## Ident аутентификация

Аналогично peer, но для TCP/IP подключений через Ident протокол (RFC 1413).

```
# pg_hba.conf
host  all  all  192.168.1.0/24  ident
```

**Редко используется:** требует запущенного Ident сервера на клиентской машине.

## LDAP аутентификация

Проверяет пароль через LDAP сервер (Active Directory, OpenLDAP и др.).

### Простой режим (simple bind)

```
# pg_hba.conf
host  all  all  0.0.0.0/0  ldap ldapserver=ldap.example.com ldapprefix="uid=" ldapsuffix=",ou=users,dc=example,dc=com"
```

Формирует DN для bind: `uid=username,ou=users,dc=example,dc=com`

### Режим поиска (search+bind)

```
# pg_hba.conf
host  all  all  0.0.0.0/0  ldap ldapserver=ldap.example.com ldapbasedn="dc=example,dc=com" ldapsearchattribute=uid ldapbinddn="cn=service,dc=example,dc=com" ldapbindpasswd=secret
```

Процесс:
1. Подключение к LDAP как service account
2. Поиск пользователя по uid
3. Bind с найденным DN и паролем пользователя

### LDAP с SSL

```
# pg_hba.conf
host  all  all  0.0.0.0/0  ldap ldapserver=ldap.example.com ldapscheme=ldaps ldapport=636 ...
```

### Полный пример для Active Directory

```
# pg_hba.conf
host  all  all  0.0.0.0/0  ldap
  ldapserver="dc1.company.com dc2.company.com"
  ldapscheme=ldaps
  ldapport=636
  ldapbasedn="OU=Users,DC=company,DC=com"
  ldapsearchattribute=sAMAccountName
  ldapbinddn="CN=PostgreSQL Service,OU=Service Accounts,DC=company,DC=com"
  ldapbindpasswd="service_password"
```

## GSSAPI/Kerberos аутентификация

Для корпоративных сред с Kerberos инфраструктурой (Active Directory).

```
# pg_hba.conf
host  all  all  0.0.0.0/0  gss include_realm=0 krb_realm=COMPANY.COM
```

**Настройка сервера:**

```bash
# 1. Создать SPN и keytab файл
# В Active Directory:
setspn -A postgres/dbserver.company.com@COMPANY.COM postgres

# 2. Экспортировать keytab
ktpass /princ postgres/dbserver.company.com@COMPANY.COM /mapuser postgresql_svc /pass * /out postgres.keytab

# 3. Настроить PostgreSQL
# postgresql.conf
krb_server_keyfile = '/path/to/postgres.keytab'
```

**Настройка клиента:**

```bash
# Получить Kerberos ticket
kinit username@COMPANY.COM

# Подключиться к PostgreSQL
psql -h dbserver.company.com -U username
```

## Certificate аутентификация

Аутентификация по SSL клиентским сертификатам.

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  cert clientcert=verify-full
```

**Настройка сервера (postgresql.conf):**

```
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'
```

**Подключение клиента:**

```bash
psql "host=dbserver sslmode=verify-full sslcert=client.crt sslkey=client.key sslrootcert=ca.crt"
```

**Маппинг CN сертификата на пользователя:**

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  cert clientcert=verify-full map=cert_map

# pg_ident.conf
cert_map  /^(.*)@company\.com$  \1
cert_map  service_account       app_user
```

## RADIUS аутентификация

Аутентификация через RADIUS сервер.

```
# pg_hba.conf
host  all  all  0.0.0.0/0  radius radiusservers="radius.company.com" radiussecrets="shared_secret" radiusports=1812
```

**Параметры:**
- `radiusservers` - список серверов через запятую
- `radiussecrets` - shared secrets для каждого сервера
- `radiusports` - порты (по умолчанию 1812)
- `radiusidentifiers` - NAS-Identifier

## PAM аутентификация

Использует Pluggable Authentication Modules для гибкой интеграции.

```
# pg_hba.conf
host  all  all  0.0.0.0/0  pam pamservice=postgresql
```

**Файл PAM сервиса (/etc/pam.d/postgresql):**

```
# Простая аутентификация через /etc/shadow
auth     required  pam_unix.so
account  required  pam_unix.so

# Или интеграция с LDAP
auth     required  pam_ldap.so
account  required  pam_ldap.so
```

## Комбинирование методов

### Разные методы для разных источников

```
# pg_hba.conf

# Локальные подключения через peer
local   all             postgres                          peer

# Репликация через сертификаты
hostssl replication     repl_user    0.0.0.0/0            cert

# Приложение через SCRAM
hostssl app_db          app_user     10.0.0.0/8           scram-sha-256

# Корпоративные пользователи через LDAP
hostssl all             all          192.168.0.0/16       ldap ldapserver=ldap.company.com ...

# Блокировка всего остального
host    all             all          0.0.0.0/0            reject
```

### Постепенный переход на SCRAM

```
# pg_hba.conf

# Новые клиенты используют SCRAM
hostssl all  all  10.1.0.0/16  scram-sha-256

# Старые клиенты пока используют md5
hostssl all  all  10.2.0.0/16  md5

# План миграции:
# 1. Обновить клиенты в 10.2.0.0/16
# 2. Изменить их метод на scram-sha-256
# 3. Обновить пароли пользователей
```

## Отладка аутентификации

### Логирование

```sql
-- postgresql.conf
log_connections = on
log_disconnections = on
log_hostname = on

-- Для детальной отладки
log_min_messages = DEBUG5
```

### Проверка метода

```bash
# Показать какой метод используется
psql -h localhost -U username -c "SELECT * FROM pg_stat_ssl"

# Проверка SCRAM
psql "host=localhost user=username password=test" 2>&1 | grep -i scram
```

### Тестирование LDAP

```bash
# Проверить LDAP подключение
ldapsearch -x -H ldaps://ldap.company.com -b "dc=company,dc=com" "uid=testuser"
```

## Best Practices

### 1. Используйте scram-sha-256 для паролей

```sql
-- postgresql.conf
password_encryption = 'scram-sha-256'
```

### 2. Всегда требуйте SSL для удалённых подключений

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  scram-sha-256  # Вместо host
```

### 3. Используйте разные методы для разных сред

```
# Локальная разработка
local   all  all  peer

# Серверы приложений
hostssl app_db  app  10.0.0.0/8  scram-sha-256

# Администраторы
hostssl all  dba  192.168.1.0/24  cert
```

### 4. Интегрируйтесь с корпоративной инфраструктурой

- Используйте LDAP/AD для централизованного управления
- Настройте SSO через Kerberos
- Используйте certificate для сервисных аккаунтов

### 5. Блокируйте нежелательные подключения

```
# В конце pg_hba.conf
host    all  all  0.0.0.0/0  reject
```

## Типичные ошибки

### 1. Trust для удалённых подключений

```
# НИКОГДА так не делайте!
host  all  all  0.0.0.0/0  trust
```

### 2. Password без SSL

```
# Пароль передаётся открытым текстом!
host  all  all  0.0.0.0/0  password

# Правильно:
hostssl  all  all  0.0.0.0/0  scram-sha-256
```

### 3. Неправильный порядок правил

```
# pg_hba.conf использует первое совпадение!

# Плохо:
host  all  all  0.0.0.0/0  reject
host  all  app  10.0.0.0/8  scram-sha-256  # Никогда не сработает!

# Правильно:
host  all  app  10.0.0.0/8  scram-sha-256
host  all  all  0.0.0.0/0  reject
```

### 4. Забыли обновить пароли при переходе на SCRAM

```sql
-- После изменения password_encryption нужно обновить пароли
ALTER USER app_user WITH PASSWORD 'password';  -- Пересоздаст хэш в новом формате
```

## Связанные темы

- [pg_hba.conf](./07-pg_hba-conf.md) - конфигурация аутентификации
- [SSL Settings](./08-ssl-settings.md) - настройка шифрования
- [Роли](./06-roles.md) - управление пользователями

---

[prev: 04-row-level-security](./04-row-level-security.md) | [next: 06-roles](./06-roles.md)
