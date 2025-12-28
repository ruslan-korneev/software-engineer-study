# Безопасность сервера

[prev: 05-csp](./05-csp.md) | [next: 07-api-security-best-practices](./07-api-security-best-practices.md)

---

## Введение

Безопасность сервера — это комплекс мер по защите серверной инфраструктуры от несанкционированного доступа, атак и утечки данных. Правильная настройка сервера критически важна для защиты приложения и данных пользователей.

## Принцип минимальных привилегий

### Пользователи и права доступа

```bash
# Создание пользователя для приложения (без sudo)
sudo adduser --system --no-create-home --group appuser

# Запуск приложения от имени непривилегированного пользователя
sudo -u appuser node /app/server.js

# Настройка прав на файлы
sudo chown -R appuser:appuser /app
sudo chmod -R 750 /app
sudo chmod 640 /app/config/*.env  # Конфиги только для чтения
```

### Systemd сервис с ограничениями

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/app

# Команда запуска
ExecStart=/usr/bin/node /app/server.js

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadOnlyPaths=/
ReadWritePaths=/app/logs /app/uploads

# Ограничение capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Сетевые ограничения
PrivateNetwork=false
RestrictAddressFamilies=AF_INET AF_INET6

# Системные вызовы
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM

# Ресурсы
MemoryMax=512M
CPUQuota=50%

[Install]
WantedBy=multi-user.target
```

## Настройка файрвола

### UFW (Ubuntu)

```bash
# Базовая настройка
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Разрешаем SSH (важно сделать до включения!)
sudo ufw allow ssh
# или конкретный порт
sudo ufw allow 22/tcp

# Разрешаем HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Ограничение SSH попыток (защита от брутфорса)
sudo ufw limit ssh

# Разрешить доступ к порту только с определённого IP
sudo ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL только из внутренней сети

# Включаем файрвол
sudo ufw enable

# Проверка статуса
sudo ufw status verbose
```

### iptables

```bash
#!/bin/bash
# Firewall script

# Очистка правил
iptables -F
iptables -X

# Политики по умолчанию
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Разрешаем loopback
iptables -A INPUT -i lo -j ACCEPT

# Разрешаем установленные соединения
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH с защитой от брутфорса
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Блокируем ICMP flooding
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s --limit-burst 4 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Защита от SYN flood
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT

# Логирование отброшенных пакетов
iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: "
```

## Защита SSH

### /etc/ssh/sshd_config

```bash
# Отключаем root логин
PermitRootLogin no

# Только ключевая аутентификация
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Ограничиваем пользователей
AllowUsers admin deploy

# Изменяем порт (опционально)
Port 2222

# Ограничиваем попытки
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30

# Отключаем ненужное
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no

# Таймаут неактивных сессий
ClientAliveInterval 300
ClientAliveCountMax 2

# Протокол
Protocol 2

# Криптография
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
```

### Fail2ban для защиты от брутфорса

```bash
# Установка
sudo apt install fail2ban

# Конфигурация /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
ignoreip = 127.0.0.1/8 10.0.0.0/8

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 3

[nginx-limit-req]
enabled = true
filter = nginx-limit-req
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 10
```

## Заголовки безопасности HTTP

### Nginx

```nginx
server {
    # HTTPS принудительно
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Защита от XSS
    add_header X-XSS-Protection "1; mode=block" always;

    # Защита от clickjacking
    add_header X-Frame-Options "DENY" always;

    # Запрет MIME sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions Policy (бывший Feature-Policy)
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    # CSP (пример)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;

    # Скрываем версию сервера
    server_tokens off;

    # Ограничение методов
    if ($request_method !~ ^(GET|HEAD|POST|PUT|DELETE)$) {
        return 405;
    }
}
```

### Express.js с Helmet

```javascript
const express = require('express');
const helmet = require('helmet');

const app = express();

// Все заголовки безопасности одной строкой
app.use(helmet());

// Или детальная настройка
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            imgSrc: ["'self'", "data:", "https:"]
        }
    },
    hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
    },
    frameguard: { action: 'deny' },
    noSniff: true,
    xssFilter: true,
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}));

// Скрываем Express
app.disable('x-powered-by');
```

## Ограничение запросов (Rate Limiting)

### Nginx

```nginx
# Определение зоны ограничения
http {
    # 10MB памяти, 10 запросов/сек на IP
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    # Ограничение соединений
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    server {
        # API endpoints
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_conn conn 10;

            proxy_pass http://backend;
        }

        # Login endpoint (более строгий)
        location /api/login {
            limit_req zone=login burst=5;

            proxy_pass http://backend;
        }
    }
}
```

### Express.js

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis();

// Общий лимит
const generalLimiter = rateLimit({
    store: new RedisStore({
        client: redis,
        prefix: 'rl:general:'
    }),
    windowMs: 15 * 60 * 1000, // 15 минут
    max: 100, // 100 запросов за окно
    message: { error: 'Too many requests, please try again later.' },
    standardHeaders: true,
    legacyHeaders: false
});

// Строгий лимит для аутентификации
const authLimiter = rateLimit({
    store: new RedisStore({
        client: redis,
        prefix: 'rl:auth:'
    }),
    windowMs: 60 * 60 * 1000, // 1 час
    max: 5, // 5 попыток
    message: { error: 'Too many login attempts, please try again later.' },
    skipSuccessfulRequests: true
});

app.use('/api/', generalLimiter);
app.use('/api/login', authLimiter);
app.use('/api/register', authLimiter);
```

## Защита от DDoS

### Nginx

```nginx
http {
    # Ограничение размера запроса
    client_max_body_size 10M;
    client_body_buffer_size 128k;

    # Таймауты
    client_body_timeout 10s;
    client_header_timeout 10s;
    send_timeout 10s;

    # Keepalive
    keepalive_timeout 30s;
    keepalive_requests 100;

    # Буферы
    large_client_header_buffers 2 1k;

    # Gzip (но осторожно с BREACH атакой)
    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain application/json;

    # Блокировка плохих User-Agent
    map $http_user_agent $bad_bot {
        default 0;
        ~*malicious 1;
        ~*scanner 1;
        ~*sqlmap 1;
    }

    server {
        if ($bad_bot) {
            return 403;
        }
    }
}
```

### Cloudflare / CDN

Для production-приложений рекомендуется использовать CDN:
- Cloudflare
- AWS CloudFront
- Fastly

## Безопасность Docker

### Dockerfile best practices

```dockerfile
# Используем минимальный базовый образ
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production образ
FROM node:20-alpine

# Создаём непривилегированного пользователя
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Копируем только необходимое
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

# Убираем лишние права
RUN chmod -R 550 /app

# Запуск от непривилегированного пользователя
USER nodejs

# Не используем root для EXPOSE
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### Docker Compose с ограничениями

```yaml
version: '3.8'

services:
  app:
    build: .
    read_only: true  # Файловая система только для чтения
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    tmpfs:
      - /tmp:noexec,nosuid,nodev
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    networks:
      - frontend
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data:rw
    networks:
      - backend
    # База данных не доступна извне
    expose:
      - "5432"

networks:
  frontend:
  backend:
    internal: true  # Изолированная сеть

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  db_data:
```

## Мониторинг и логирование

### Логирование безопасности

```python
# Python logging конфигурация
import logging
from logging.handlers import RotatingFileHandler
import json

class SecurityLogger:
    def __init__(self):
        self.logger = logging.getLogger('security')
        self.logger.setLevel(logging.INFO)

        handler = RotatingFileHandler(
            '/var/log/app/security.log',
            maxBytes=10*1024*1024,  # 10MB
            backupCount=10
        )

        formatter = logging.Formatter(
            '%(asctime)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)

    def log_event(self, event_type, user_id=None, ip=None, details=None):
        event = {
            'type': event_type,
            'user_id': user_id,
            'ip': ip,
            'details': details
        }
        self.logger.info(json.dumps(event))

# Использование
security_log = SecurityLogger()

@app.route('/login', methods=['POST'])
def login():
    ip = request.remote_addr
    username = request.form.get('username')

    if authenticate(username, request.form.get('password')):
        security_log.log_event('LOGIN_SUCCESS', user_id=username, ip=ip)
        return redirect('/dashboard')
    else:
        security_log.log_event('LOGIN_FAILED', ip=ip, details={'username': username})
        return 'Invalid credentials', 401
```

## Чек-лист безопасности сервера

### Базовая настройка
- [ ] Обновления установлены (apt update && apt upgrade)
- [ ] Автоматические обновления безопасности включены
- [ ] Ненужные сервисы отключены
- [ ] Файрвол настроен (UFW/iptables)

### SSH
- [ ] Root логин отключён
- [ ] Парольная аутентификация отключена
- [ ] Fail2ban настроен
- [ ] SSH порт изменён (опционально)

### Приложение
- [ ] Запуск от непривилегированного пользователя
- [ ] Заголовки безопасности настроены
- [ ] Rate limiting включён
- [ ] Логирование безопасности настроено

### Мониторинг
- [ ] Централизованные логи
- [ ] Алерты на подозрительную активность
- [ ] Регулярные security-аудиты

## Заключение

Безопасность сервера — многоуровневая задача. Основные принципы:

1. **Минимальные привилегии** — каждый процесс имеет только необходимые права
2. **Защита в глубину** — несколько уровней защиты
3. **Регулярные обновления** — патчи безопасности критичны
4. **Мониторинг** — обнаружение атак в реальном времени
5. **Автоматизация** — безопасность должна быть в CI/CD

---

[prev: 05-csp](./05-csp.md) | [next: 07-api-security-best-practices](./07-api-security-best-practices.md)
