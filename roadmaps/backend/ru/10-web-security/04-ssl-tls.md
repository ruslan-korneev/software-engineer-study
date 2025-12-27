# SSL/TLS — Протоколы безопасной передачи данных

## Что такое SSL и TLS?

**SSL** (Secure Sockets Layer) и **TLS** (Transport Layer Security) — это криптографические протоколы, обеспечивающие безопасную передачу данных в сети. TLS является современным развитием SSL.

### История версий

| Протокол | Год | Статус |
|----------|-----|--------|
| SSL 1.0 | 1994 | Никогда не был опубликован |
| SSL 2.0 | 1995 | Устарел (небезопасен) |
| SSL 3.0 | 1996 | Устарел (POODLE-атака) |
| TLS 1.0 | 1999 | Устарел |
| TLS 1.1 | 2006 | Устарел |
| TLS 1.2 | 2008 | Рекомендуется |
| TLS 1.3 | 2018 | Рекомендуется (самый безопасный) |

**Важно:** Используйте только TLS 1.2 и TLS 1.3 в production!

## Как работает TLS?

### TLS Handshake (Рукопожатие)

TLS Handshake устанавливает защищённое соединение между клиентом и сервером.

#### TLS 1.2 Handshake

```
Клиент                                            Сервер
   |                                                  |
   |--- ClientHello ------------------------------>  |
   |    Версия TLS, случайные данные,                |
   |    поддерживаемые шифры                         |
   |                                                  |
   |<-- ServerHello -------------------------------- |
   |<-- Certificate -------------------------------- |
   |<-- ServerKeyExchange -------------------------- |
   |<-- ServerHelloDone ---------------------------- |
   |                                                  |
   |--- ClientKeyExchange -------------------------> |
   |--- ChangeCipherSpec -------------------------->  |
   |--- Finished ----------------------------------> |
   |                                                  |
   |<-- ChangeCipherSpec --------------------------- |
   |<-- Finished ----------------------------------- |
   |                                                  |
   |<========= Защищённое соединение =============>  |
```

#### TLS 1.3 Handshake (Упрощённый)

TLS 1.3 требует всего **1 RTT** (Round-Trip Time) вместо 2:

```
Клиент                                            Сервер
   |                                                  |
   |--- ClientHello + KeyShare ------------------>   |
   |                                                  |
   |<-- ServerHello + KeyShare ------------------- |
   |<-- EncryptedExtensions ---------------------- |
   |<-- Certificate ------------------------------ |
   |<-- CertificateVerify ------------------------ |
   |<-- Finished --------------------------------- |
   |                                                  |
   |--- Finished ----------------------------------> |
   |                                                  |
   |<========= Защищённое соединение =============>  |
```

## Компоненты TLS

### 1. Сертификаты X.509

Сертификат содержит:
- Открытый ключ сервера
- Информацию о владельце
- Цифровую подпись Certificate Authority (CA)
- Срок действия

```bash
# Просмотр сертификата
openssl x509 -in certificate.crt -text -noout

# Проверка цепочки сертификатов
openssl verify -CAfile ca_bundle.crt certificate.crt

# Получение сертификата с сервера
openssl s_client -connect example.com:443 -servername example.com
```

### 2. Cipher Suites (Наборы шифров)

Cipher Suite определяет алгоритмы для:
- **Обмен ключами**: ECDHE, DHE, RSA
- **Аутентификация**: RSA, ECDSA
- **Шифрование**: AES-GCM, ChaCha20
- **MAC/хэширование**: SHA256, SHA384

**Пример TLS 1.2 cipher suite:**
```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
     │      │        │        │
     │      │        │        └── Хэширование
     │      │        └── Симметричное шифрование
     │      └── Аутентификация
     └── Обмен ключами
```

**TLS 1.3 cipher suites (упрощённые):**
```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

## Практические примеры настройки

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # Сертификаты
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Современные протоколы
    ssl_protocols TLSv1.2 TLSv1.3;

    # Безопасные шифры для TLS 1.2
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;

    # Предпочтение серверных шифров
    ssl_prefer_server_ciphers on;

    # Параметры Diffie-Hellman
    ssl_dhparam /etc/ssl/dhparam.pem;

    # Сессионный кэш
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;  # Для Perfect Forward Secrecy

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca_bundle.crt;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

### Apache

```apache
<VirtualHost *:443>
    ServerName example.com

    # SSL Engine
    SSLEngine on

    # Сертификаты
    SSLCertificateFile /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/ca_bundle.crt

    # Протоколы
    SSLProtocol -all +TLSv1.2 +TLSv1.3

    # Шифры
    SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

    # Предпочтение серверных шифров
    SSLHonorCipherOrder on

    # OCSP Stapling
    SSLUseStapling on
    SSLStaplingResponderTimeout 5
    SSLStaplingReturnResponderErrors off

    # HSTS
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
</VirtualHost>

# Глобальный кэш OCSP
SSLStaplingCache shmcb:/var/run/ocsp(128000)
```

### Node.js

```javascript
const https = require('https');
const fs = require('fs');
const tls = require('tls');

const options = {
    // Сертификаты
    key: fs.readFileSync('/path/to/private.key'),
    cert: fs.readFileSync('/path/to/certificate.crt'),
    ca: fs.readFileSync('/path/to/ca_bundle.crt'),

    // Минимальная версия TLS
    minVersion: 'TLSv1.2',

    // Предпочитаемая версия
    maxVersion: 'TLSv1.3',

    // Шифры для TLS 1.2
    ciphers: [
        'ECDHE-ECDSA-AES128-GCM-SHA256',
        'ECDHE-RSA-AES128-GCM-SHA256',
        'ECDHE-ECDSA-AES256-GCM-SHA384',
        'ECDHE-RSA-AES256-GCM-SHA384'
    ].join(':'),

    // Предпочтение серверных шифров
    honorCipherOrder: true,

    // Session tickets (для PFS лучше отключить)
    secureOptions: require('constants').SSL_OP_NO_TICKET
};

const server = https.createServer(options, (req, res) => {
    res.writeHead(200);
    res.end('Secure connection!');
});

server.listen(443);
```

### Python (Flask/Gunicorn)

```python
# gunicorn.conf.py
import ssl

bind = "0.0.0.0:443"
workers = 4

# SSL настройки
keyfile = "/path/to/private.key"
certfile = "/path/to/certificate.crt"
ca_certs = "/path/to/ca_bundle.crt"

# Минимальная версия TLS
ssl_version = ssl.TLSVersion.TLSv1_2

# Шифры
ciphers = "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
```

```python
# Python ssl context
import ssl

context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.minimum_version = ssl.TLSVersion.TLSv1_2
context.maximum_version = ssl.TLSVersion.TLSv1_3
context.load_cert_chain('certificate.crt', 'private.key')
context.set_ciphers('ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256')
```

## Генерация сертификатов

### Самоподписанный сертификат (для тестирования)

```bash
# Генерация приватного ключа и сертификата
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key \
    -out certificate.crt \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=Company/CN=localhost"

# С поддержкой SAN (Subject Alternative Names)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key \
    -out certificate.crt \
    -config <(cat << EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
x509_extensions = v3_req

[dn]
C = RU
ST = Moscow
L = Moscow
O = Company
CN = localhost

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = *.localhost
IP.1 = 127.0.0.1
EOF
)
```

### CSR для CA-подписанного сертификата

```bash
# 1. Генерация приватного ключа
openssl genrsa -out private.key 2048

# 2. Создание CSR (Certificate Signing Request)
openssl req -new -key private.key -out request.csr \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=Company/CN=example.com"

# 3. Отправка CSR в Certificate Authority
# 4. Получение сертификата от CA
```

### Let's Encrypt (Certbot)

```bash
# Установка
sudo apt install certbot python3-certbot-nginx

# Получение сертификата
sudo certbot --nginx -d example.com -d www.example.com

# Автоматическое обновление
sudo certbot renew --dry-run

# Добавление в cron
0 0 1 * * /usr/bin/certbot renew --quiet
```

## Проверка и тестирование

### Командная строка

```bash
# Проверка соединения
openssl s_client -connect example.com:443 -servername example.com

# Проверка поддерживаемых протоколов
nmap --script ssl-enum-ciphers -p 443 example.com

# Проверка сертификата
openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Проверка TLS 1.3
openssl s_client -connect example.com:443 -tls1_3
```

### Онлайн-инструменты

- **SSL Labs**: https://www.ssllabs.com/ssltest/
- **Hardenize**: https://www.hardenize.com/
- **Security Headers**: https://securityheaders.com/

### Python-скрипт для проверки

```python
import ssl
import socket

def check_ssl(hostname, port=443):
    context = ssl.create_default_context()

    with socket.create_connection((hostname, port)) as sock:
        with context.wrap_socket(sock, server_hostname=hostname) as ssock:
            print(f"TLS версия: {ssock.version()}")
            print(f"Шифр: {ssock.cipher()}")

            cert = ssock.getpeercert()
            print(f"Издатель: {dict(x[0] for x in cert['issuer'])}")
            print(f"Действителен до: {cert['notAfter']}")

check_ssl('example.com')
```

## Perfect Forward Secrecy (PFS)

**PFS** гарантирует, что компрометация долгосрочного ключа не позволит расшифровать ранее записанный трафик.

### Как работает PFS

```
Без PFS (RSA):
┌─────────────┐        ┌─────────────┐
│   Клиент    │        │   Сервер    │
└──────┬──────┘        └──────┬──────┘
       │                       │
       │ Premaster secret,     │
       │ зашифрованный RSA --> │  Если приватный ключ
       │                       │  украден, весь прошлый
       │<-- Данные ----------->│  трафик расшифровывается
       │                       │

С PFS (ECDHE):
┌─────────────┐        ┌─────────────┐
│   Клиент    │        │   Сервер    │
└──────┬──────┘        └──────┬──────┘
       │                       │
       │<-- Ephemeral DH --->  │  Каждая сессия использует
       │     параметры         │  новый временный ключ
       │                       │
       │<-- Данные ----------->│  Компрометация RSA-ключа
       │                       │  не поможет расшифровать
```

### Включение PFS

```nginx
# Nginx - используйте шифры с ECDHE
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers on;

# Отключите session tickets для строгого PFS
ssl_session_tickets off;
```

## Типичные ошибки

### 1. Использование устаревших протоколов

```nginx
# ПЛОХО
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;

# ХОРОШО
ssl_protocols TLSv1.2 TLSv1.3;
```

### 2. Слабые шифры

```nginx
# ПЛОХО (включает слабые шифры)
ssl_ciphers ALL:!aNULL:!MD5;

# ХОРОШО
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
```

### 3. Отсутствие HSTS

```nginx
# Обязательно добавьте
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### 4. Неполная цепочка сертификатов

```bash
# Проверка цепочки
openssl s_client -connect example.com:443 -servername example.com 2>&1 | grep -i verify
```

## Заключение

SSL/TLS — основа безопасной передачи данных в интернете. Ключевые рекомендации:

1. **Используйте только TLS 1.2 и 1.3**
2. **Настройте сильные cipher suites с PFS (ECDHE)**
3. **Включите HSTS для принудительного HTTPS**
4. **Регулярно обновляйте сертификаты**
5. **Проверяйте конфигурацию через SSL Labs**
6. **Используйте OCSP Stapling для производительности**
