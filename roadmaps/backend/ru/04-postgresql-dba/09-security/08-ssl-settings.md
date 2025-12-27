# Настройки SSL в PostgreSQL

## Введение

SSL (Secure Sockets Layer) / TLS (Transport Layer Security) обеспечивает шифрование данных при передаче между клиентом и сервером PostgreSQL. Это критически важно для защиты конфиденциальных данных, особенно при подключении через ненадёжные сети.

## Зачем нужен SSL

- **Шифрование трафика**: данные не могут быть перехвачены
- **Аутентификация сервера**: клиент уверен, что подключается к правильному серверу
- **Аутентификация клиента**: сервер может проверить клиента по сертификату
- **Целостность данных**: защита от модификации данных в пути
- **Compliance**: требуется стандартами PCI DSS, HIPAA, GDPR

## Создание сертификатов

### Самоподписанный сертификат (для разработки)

```bash
# Создание директории
mkdir -p /var/lib/postgresql/ssl
cd /var/lib/postgresql/ssl

# Генерация приватного ключа сервера
openssl genrsa -out server.key 4096

# Установка правильных прав
chmod 600 server.key
chown postgres:postgres server.key

# Создание самоподписанного сертификата (10 лет)
openssl req -new -x509 -days 3650 -key server.key -out server.crt \
    -subj "/CN=postgresql-server/O=MyCompany/C=RU"

# Установка прав
chmod 644 server.crt
chown postgres:postgres server.crt
```

### Создание CA и сертификатов (для production)

```bash
# === 1. Создание корневого CA ===
# Приватный ключ CA
openssl genrsa -aes256 -out ca.key 4096

# Сертификат CA (10 лет)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
    -subj "/CN=PostgreSQL-CA/O=MyCompany/C=RU"

# === 2. Создание сертификата сервера ===
# Приватный ключ сервера
openssl genrsa -out server.key 4096

# Запрос на подпись (CSR)
openssl req -new -key server.key -out server.csr \
    -subj "/CN=db.example.com/O=MyCompany/C=RU"

# Файл расширений для сервера
cat > server.ext << EOF
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "PostgreSQL Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = db.example.com
DNS.2 = db
DNS.3 = localhost
IP.1 = 10.0.0.100
IP.2 = 127.0.0.1
EOF

# Подпись сертификата CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out server.crt -days 365 \
    -extfile server.ext

# === 3. Создание клиентского сертификата ===
# Приватный ключ клиента
openssl genrsa -out client.key 4096

# Запрос на подпись (CSR) - CN должен совпадать с именем пользователя PostgreSQL
openssl req -new -key client.key -out client.csr \
    -subj "/CN=app_user/O=MyCompany/C=RU"

# Файл расширений для клиента
cat > client.ext << EOF
basicConstraints = CA:FALSE
nsCertType = client
nsComment = "PostgreSQL Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF

# Подпись сертификата CA
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out client.crt -days 365 \
    -extfile client.ext

# === 4. Установка прав ===
chmod 600 server.key client.key ca.key
chmod 644 server.crt client.crt ca.crt
chown postgres:postgres server.key server.crt ca.crt
```

### Проверка сертификатов

```bash
# Информация о сертификате
openssl x509 -in server.crt -text -noout

# Проверка соответствия ключа и сертификата
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
# Хэши должны совпадать

# Проверка цепочки сертификатов
openssl verify -CAfile ca.crt server.crt
openssl verify -CAfile ca.crt client.crt

# Проверка срока действия
openssl x509 -in server.crt -noout -dates
```

## Конфигурация сервера

### Базовые настройки (postgresql.conf)

```
# === SSL Configuration ===

# Включение SSL
ssl = on

# Пути к сертификатам (относительно data_directory или абсолютные)
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'

# CA для проверки клиентских сертификатов
ssl_ca_file = 'ca.crt'

# Список отозванных сертификатов (опционально)
ssl_crl_file = 'root.crl'
ssl_crl_dir = ''

# Директория для CA сертификатов
# ssl_ca_dir = ''
```

### Настройки безопасности

```
# Минимальная версия TLS (рекомендуется TLSv1.2 или TLSv1.3)
ssl_min_protocol_version = 'TLSv1.2'
ssl_max_protocol_version = ''  # Не ограничивать максимум

# Разрешённые шифры (TLS 1.2)
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# Для максимальной безопасности
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305'

# Предпочитать серверные шифры
ssl_prefer_server_ciphers = on

# Параметры DH для обмена ключами
ssl_dh_params_file = ''  # Или путь к dhparam.pem

# ECDH кривая
ssl_ecdh_curve = 'prime256v1'

# Пароль для приватного ключа (если зашифрован)
ssl_passphrase_command = ''
ssl_passphrase_command_supports_reload = off
```

### Генерация DH параметров

```bash
# Генерация DH параметров (занимает время)
openssl dhparam -out dhparam.pem 2048

# Использование в postgresql.conf
ssl_dh_params_file = '/var/lib/postgresql/ssl/dhparam.pem'
```

## Конфигурация pg_hba.conf

```
# Требовать SSL для всех удалённых подключений
hostssl  all             all             0.0.0.0/0              scram-sha-256

# SSL с проверкой клиентского сертификата
hostssl  all             all             10.0.0.0/8             cert

# Дополнительная проверка: clientcert + пароль
hostssl  all             all             192.168.0.0/16         scram-sha-256 clientcert=verify-full

# Запретить не-SSL подключения
hostnossl all            all             0.0.0.0/0              reject
```

### Опции clientcert

| Опция | Описание |
|-------|----------|
| `clientcert=verify-ca` | Проверить, что сертификат подписан CA |
| `clientcert=verify-full` | verify-ca + проверить CN сертификата = имени пользователя |
| `clientcert=0` или `no-verify` | Не требовать сертификат (по умолчанию) |

## Конфигурация клиента

### Режимы SSL (sslmode)

| Режим | Описание |
|-------|----------|
| `disable` | Без SSL |
| `allow` | Сначала без SSL, если не получится - с SSL |
| `prefer` | Сначала с SSL, если не получится - без SSL (по умолчанию) |
| `require` | Только SSL, не проверять сертификат сервера |
| `verify-ca` | SSL + проверка, что сертификат подписан CA |
| `verify-full` | verify-ca + проверка имени хоста в сертификате |

### Подключение через psql

```bash
# Простое SSL подключение
psql "host=db.example.com dbname=mydb user=myuser sslmode=require"

# С проверкой сертификата сервера
psql "host=db.example.com dbname=mydb user=myuser sslmode=verify-full sslrootcert=/path/to/ca.crt"

# С клиентским сертификатом
psql "host=db.example.com dbname=mydb user=myuser \
  sslmode=verify-full \
  sslcert=/path/to/client.crt \
  sslkey=/path/to/client.key \
  sslrootcert=/path/to/ca.crt"
```

### Переменные окружения

```bash
export PGSSLMODE=verify-full
export PGSSLCERT=/path/to/client.crt
export PGSSLKEY=/path/to/client.key
export PGSSLROOTCERT=/path/to/ca.crt

psql -h db.example.com -U myuser -d mydb
```

### Файл ~/.postgresql/

```bash
# Расположение по умолчанию для клиентских сертификатов
~/.postgresql/postgresql.crt    # Клиентский сертификат
~/.postgresql/postgresql.key    # Приватный ключ (права 600!)
~/.postgresql/root.crt          # CA сертификат для проверки сервера
~/.postgresql/root.crl          # Список отзыва
```

### Настройка в connection string приложения

```python
# Python с psycopg2
import psycopg2

conn = psycopg2.connect(
    host="db.example.com",
    database="mydb",
    user="app_user",
    password="password",
    sslmode="verify-full",
    sslcert="/path/to/client.crt",
    sslkey="/path/to/client.key",
    sslrootcert="/path/to/ca.crt"
)
```

```javascript
// Node.js с pg
const { Client } = require('pg');

const client = new Client({
  host: 'db.example.com',
  database: 'mydb',
  user: 'app_user',
  password: 'password',
  ssl: {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/path/to/ca.crt').toString(),
    key: fs.readFileSync('/path/to/client.key').toString(),
    cert: fs.readFileSync('/path/to/client.crt').toString(),
  }
});
```

```java
// Java JDBC
String url = "jdbc:postgresql://db.example.com/mydb?" +
    "ssl=true&" +
    "sslmode=verify-full&" +
    "sslcert=/path/to/client.crt&" +
    "sslkey=/path/to/client.key&" +
    "sslrootcert=/path/to/ca.crt";
```

## Мониторинг SSL подключений

### Просмотр SSL статуса

```sql
-- Информация о текущем подключении
SELECT
    ssl,
    version AS ssl_version,
    cipher,
    bits,
    client_dn,
    client_serial,
    issuer_dn
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();

-- Все SSL подключения
SELECT
    a.pid,
    a.usename,
    a.client_addr,
    a.application_name,
    s.ssl,
    s.version,
    s.cipher,
    s.bits,
    s.client_dn
FROM pg_stat_activity a
JOIN pg_stat_ssl s ON a.pid = s.pid
WHERE a.state IS NOT NULL;

-- Статистика по SSL
SELECT
    ssl,
    COUNT(*) as connections,
    array_agg(DISTINCT version) as ssl_versions,
    array_agg(DISTINCT cipher) as ciphers
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE state IS NOT NULL
GROUP BY ssl;
```

### Проверка конфигурации

```sql
-- Текущие SSL настройки
SELECT name, setting, unit, context
FROM pg_settings
WHERE name LIKE 'ssl%';

-- Проверка сертификата
-- (требует superuser или pg_read_server_files)
SELECT pg_read_file('server.crt');
```

## Обновление сертификатов

### Перезагрузка без рестарта

```sql
-- Перезагрузка конфигурации (включая SSL)
SELECT pg_reload_conf();

-- PostgreSQL перезагрузит сертификаты при следующем подключении
-- Существующие подключения продолжат работать со старыми сертификатами
```

### Автоматизация обновления (Let's Encrypt)

```bash
#!/bin/bash
# certbot-postgres-hook.sh

# Копирование сертификатов
cp /etc/letsencrypt/live/db.example.com/fullchain.pem /var/lib/postgresql/ssl/server.crt
cp /etc/letsencrypt/live/db.example.com/privkey.pem /var/lib/postgresql/ssl/server.key

# Установка прав
chown postgres:postgres /var/lib/postgresql/ssl/server.*
chmod 600 /var/lib/postgresql/ssl/server.key
chmod 644 /var/lib/postgresql/ssl/server.crt

# Перезагрузка PostgreSQL
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

```bash
# Добавление в certbot
certbot certonly --standalone -d db.example.com \
    --deploy-hook /usr/local/bin/certbot-postgres-hook.sh
```

## CRL - Список отозванных сертификатов

### Создание CRL

```bash
# Отзыв сертификата
openssl ca -revoke client.crt -keyfile ca.key -cert ca.crt

# Генерация CRL
openssl ca -gencrl -keyfile ca.key -cert ca.crt -out root.crl

# Для простого случая (без полноценной CA)
openssl crl -in root.crl -text -noout
```

### Настройка PostgreSQL

```
# postgresql.conf
ssl_crl_file = 'root.crl'

# Или директория с CRL
ssl_crl_dir = '/var/lib/postgresql/ssl/crl'
```

## Best Practices

### 1. Всегда используйте verify-full на клиенте

```bash
# В production всегда проверяйте сервер
PGSSLMODE=verify-full psql ...
```

### 2. Используйте современные протоколы

```
# postgresql.conf
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'
```

### 3. Используйте клиентские сертификаты для сервисов

```
# pg_hba.conf
hostssl  all  service_user  10.0.0.0/8  cert clientcert=verify-full
```

### 4. Мониторьте срок действия сертификатов

```bash
#!/bin/bash
# check-cert-expiry.sh
CERT="/var/lib/postgresql/ssl/server.crt"
WARN_DAYS=30

expiry=$(openssl x509 -enddate -noout -in "$CERT" | cut -d= -f2)
expiry_epoch=$(date -d "$expiry" +%s)
now_epoch=$(date +%s)
days_left=$(( ($expiry_epoch - $now_epoch) / 86400 ))

if [ $days_left -lt $WARN_DAYS ]; then
    echo "WARNING: Certificate expires in $days_left days"
    exit 1
fi
```

### 5. Разделяйте ключи по средам

```
# Разные CA для разных окружений
production-ca.crt
staging-ca.crt
development-ca.crt
```

### 6. Храните приватные ключи безопасно

```bash
# Правильные права
chmod 600 server.key
chown postgres:postgres server.key

# Шифрование ключа паролем
openssl rsa -aes256 -in server.key -out server.key.encrypted

# postgresql.conf
ssl_key_file = 'server.key.encrypted'
ssl_passphrase_command = 'echo "password"'  # Или более безопасный метод
```

## Типичные ошибки

### 1. Неправильные права на ключ

```
LOG:  could not load private key file "server.key": Permission denied
FATAL:  could not access private key file "server.key"

# Решение
chmod 600 server.key
chown postgres:postgres server.key
```

### 2. Несоответствие сертификата и ключа

```
LOG:  SSL error: key values mismatch

# Проверка
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
# Должны совпадать
```

### 3. Истёкший сертификат

```
SSL error: certificate has expired

# Проверка
openssl x509 -in server.crt -noout -dates
```

### 4. Неправильное имя хоста в verify-full

```
SSL error: certificate verify failed (hostname mismatch)

# Проверка
openssl x509 -in server.crt -noout -text | grep -A1 "Subject Alternative Name"
# Должен содержать имя хоста или IP
```

### 5. Отсутствует CA сертификат

```
SSL error: unable to get local issuer certificate

# Решение: указать CA сертификат
psql "... sslrootcert=/path/to/ca.crt"
```

## Отладка

```bash
# Проверка SSL соединения
openssl s_client -connect db.example.com:5432 -starttls postgres

# С проверкой сертификата
openssl s_client -connect db.example.com:5432 -starttls postgres \
    -CAfile ca.crt -verify 5

# Детальная информация
openssl s_client -connect db.example.com:5432 -starttls postgres \
    -CAfile ca.crt -verify 5 -state -debug

# Проверка поддерживаемых шифров
nmap --script ssl-enum-ciphers -p 5432 db.example.com
```

## Связанные темы

- [pg_hba.conf](./07-pg_hba-conf.md) - настройка аутентификации
- [Модели аутентификации](./05-authentication-models.md) - методы аутентификации
- [Роли](./06-roles.md) - управление пользователями
