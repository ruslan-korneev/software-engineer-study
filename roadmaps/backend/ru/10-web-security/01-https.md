# HTTPS (HyperText Transfer Protocol Secure)

[prev: 05-memcached](../09-caching/05-memcached.md) | [next: 02-owasp-risks](./02-owasp-risks.md)

---

## Что такое HTTPS?

HTTPS — это защищённая версия протокола HTTP, которая обеспечивает шифрование данных между браузером пользователя и веб-сервером. HTTPS использует криптографические протоколы SSL/TLS для защиты передаваемой информации.

## Зачем нужен HTTPS?

### 1. Конфиденциальность
Все данные шифруются, что предотвращает их перехват злоумышленниками (атаки "man-in-the-middle").

### 2. Целостность данных
HTTPS гарантирует, что данные не были изменены во время передачи.

### 3. Аутентификация
SSL-сертификат подтверждает, что пользователь общается именно с тем сервером, с которым намеревался.

### 4. SEO и доверие
- Google отдаёт приоритет HTTPS-сайтам в поисковой выдаче
- Браузеры помечают HTTP-сайты как "небезопасные"

## Как работает HTTPS?

### Процесс установления соединения (TLS Handshake)

```
Клиент                              Сервер
   |                                   |
   |--- ClientHello ------------------>|  1. Клиент отправляет версию TLS,
   |                                   |     поддерживаемые шифры
   |                                   |
   |<-- ServerHello -------------------|  2. Сервер выбирает параметры,
   |<-- Certificate -------------------|     отправляет сертификат
   |<-- ServerHelloDone ---------------|
   |                                   |
   |--- ClientKeyExchange ------------>|  3. Клиент генерирует и отправляет
   |--- ChangeCipherSpec ------------->|     ключ сессии
   |--- Finished --------------------->|
   |                                   |
   |<-- ChangeCipherSpec --------------|  4. Сервер подтверждает
   |<-- Finished ----------------------|
   |                                   |
   |<== Зашифрованное соединение =====>|  5. Начинается обмен данными
```

## Практические примеры

### Настройка HTTPS в Node.js (Express)

```javascript
const https = require('https');
const fs = require('fs');
const express = require('express');

const app = express();

// Загрузка SSL-сертификатов
const options = {
    key: fs.readFileSync('/path/to/private.key'),
    cert: fs.readFileSync('/path/to/certificate.crt'),
    // Опционально: CA-сертификат для цепочки доверия
    ca: fs.readFileSync('/path/to/ca_bundle.crt')
};

app.get('/', (req, res) => {
    res.send('Безопасное соединение установлено!');
});

// Запуск HTTPS-сервера
https.createServer(options, app).listen(443, () => {
    console.log('HTTPS сервер запущен на порту 443');
});

// Редирект с HTTP на HTTPS
const http = require('http');
http.createServer((req, res) => {
    res.writeHead(301, {
        'Location': `https://${req.headers.host}${req.url}`
    });
    res.end();
}).listen(80);
```

### Настройка HTTPS в Python (Flask)

```python
from flask import Flask, redirect, request
import ssl

app = Flask(__name__)

@app.route('/')
def index():
    return 'Безопасное соединение!'

# Принудительный редирект на HTTPS
@app.before_request
def force_https():
    if not request.is_secure:
        url = request.url.replace('http://', 'https://', 1)
        return redirect(url, code=301)

if __name__ == '__main__':
    # SSL-контекст
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.load_cert_chain('certificate.crt', 'private.key')

    app.run(host='0.0.0.0', port=443, ssl_context=context)
```

### Настройка HTTPS в Nginx

```nginx
# Редирект HTTP -> HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS-сервер
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL-сертификаты
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Современные настройки безопасности
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;

    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling для быстрой проверки сертификата
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Получение SSL-сертификата

### Let's Encrypt (бесплатно)

```bash
# Установка Certbot
sudo apt install certbot python3-certbot-nginx

# Получение сертификата для Nginx
sudo certbot --nginx -d example.com -d www.example.com

# Автоматическое обновление (добавить в cron)
sudo certbot renew --dry-run
```

### Типы сертификатов

| Тип | Описание | Цена |
|-----|----------|------|
| **DV** (Domain Validation) | Подтверждает только домен | Бесплатно - $ |
| **OV** (Organization Validation) | Подтверждает организацию | $$ |
| **EV** (Extended Validation) | Расширенная проверка | $$$ |
| **Wildcard** | Для всех поддоменов (*.example.com) | $-$$ |

## Лучшие практики

### 1. Используйте современные протоколы
```nginx
# Только TLS 1.2 и 1.3
ssl_protocols TLSv1.2 TLSv1.3;
```

### 2. Включите HSTS
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
```

### 3. Настройте правильные заголовки
```javascript
// Express.js с helmet
const helmet = require('helmet');
app.use(helmet());

// Ручная настройка
app.use((req, res, next) => {
    res.setHeader('Strict-Transport-Security', 'max-age=31536000');
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    next();
});
```

## Типичные ошибки

### 1. Mixed Content
```html
<!-- ОШИБКА: загрузка HTTP-ресурсов на HTTPS-странице -->
<img src="http://example.com/image.jpg">

<!-- ПРАВИЛЬНО: используйте HTTPS или относительные протоколы -->
<img src="https://example.com/image.jpg">
<img src="//example.com/image.jpg">
```

### 2. Неправильная конфигурация сертификатов
```bash
# Проверка сертификата
openssl s_client -connect example.com:443 -servername example.com

# Проверка цепочки сертификатов
openssl verify -CAfile ca_bundle.crt certificate.crt
```

### 3. Отсутствие редиректа с HTTP
Всегда настраивайте автоматический редирект с HTTP на HTTPS.

## Проверка безопасности

### Онлайн-инструменты
- [SSL Labs](https://www.ssllabs.com/ssltest/) — детальный анализ конфигурации
- [Security Headers](https://securityheaders.com/) — проверка заголовков безопасности

### Командная строка
```bash
# Проверка сертификата
curl -vI https://example.com

# Детальная информация о TLS
openssl s_client -connect example.com:443

# Проверка поддерживаемых протоколов
nmap --script ssl-enum-ciphers -p 443 example.com
```

## Заключение

HTTPS — это обязательный стандарт для современных веб-приложений. Он обеспечивает:
- Защиту данных пользователей
- Доверие к вашему сайту
- Лучшие позиции в поисковых системах

Всегда используйте актуальные версии TLS (1.2+), регулярно обновляйте сертификаты и следите за рекомендациями по безопасности.

---

[prev: 05-memcached](../09-caching/05-memcached.md) | [next: 02-owasp-risks](./02-owasp-risks.md)
