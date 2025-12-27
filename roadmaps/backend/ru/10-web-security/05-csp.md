# Content Security Policy (CSP)

## Что такое CSP?

**Content Security Policy (CSP)** — это механизм безопасности, который позволяет веб-сайтам контролировать, какие ресурсы могут загружаться и выполняться на странице. CSP является мощной защитой от XSS-атак (Cross-Site Scripting) и других атак с внедрением кода.

## Как работает CSP?

CSP устанавливается через HTTP-заголовок или meta-тег. Браузер проверяет каждый загружаемый ресурс на соответствие политике и блокирует нарушения.

### Пример работы

```
┌─────────────┐     Запрос страницы      ┌─────────────┐
│   Браузер   │ -----------------------> │   Сервер    │
└─────────────┘                          └─────────────┘
      ▲                                        │
      │         Ответ с заголовком:            │
      │         Content-Security-Policy:       │
      └─────── script-src 'self' ─────────────┘

Браузер загружает страницу и применяет политику:
✓ <script src="/app.js">          — разрешено ('self')
✗ <script src="http://evil.com">  — заблокировано
✗ <script>alert('xss')</script>   — заблокировано (inline)
```

## Синтаксис CSP

### Структура директивы

```
Content-Security-Policy: директива источник1 источник2; директива2 источник;
```

### Основные директивы

| Директива | Описание |
|-----------|----------|
| `default-src` | Политика по умолчанию для всех типов ресурсов |
| `script-src` | Источники JavaScript |
| `style-src` | Источники CSS |
| `img-src` | Источники изображений |
| `font-src` | Источники шрифтов |
| `connect-src` | Разрешённые URL для fetch, XHR, WebSocket |
| `media-src` | Источники audio и video |
| `object-src` | Источники для `<object>`, `<embed>`, `<applet>` |
| `frame-src` | Источники для iframe |
| `frame-ancestors` | Какие страницы могут встраивать эту страницу |
| `form-action` | Разрешённые URL для отправки форм |
| `base-uri` | Ограничение для `<base>` тега |
| `report-uri` / `report-to` | Куда отправлять отчёты о нарушениях |

### Значения источников

| Значение | Описание | Пример |
|----------|----------|--------|
| `'self'` | Текущий origin | `script-src 'self'` |
| `'none'` | Ничего не разрешено | `object-src 'none'` |
| `'unsafe-inline'` | Разрешает inline-код | `script-src 'unsafe-inline'` |
| `'unsafe-eval'` | Разрешает eval() | `script-src 'unsafe-eval'` |
| `'strict-dynamic'` | Доверие к динамически загруженным скриптам | С nonce/hash |
| `'nonce-xxx'` | Разрешение по уникальному токену | `script-src 'nonce-abc123'` |
| `'sha256-xxx'` | Разрешение по хэшу контента | `script-src 'sha256-...'` |
| `https:` | Только HTTPS-источники | `img-src https:` |
| `data:` | Data URL | `img-src data:` |
| `blob:` | Blob URL | `worker-src blob:` |
| `*.example.com` | Wildcard для поддоменов | `script-src *.cdn.com` |

## Практические примеры

### Базовая политика

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://api.example.com; frame-ancestors 'none'; form-action 'self'; base-uri 'self'; object-src 'none'
```

### Настройка в Node.js (Express)

```javascript
const express = require('express');
const helmet = require('helmet');
const crypto = require('crypto');

const app = express();

// Генерация nonce для каждого запроса
app.use((req, res, next) => {
    res.locals.nonce = crypto.randomBytes(16).toString('base64');
    next();
});

// CSP с helmet
app.use(
    helmet.contentSecurityPolicy({
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: [
                "'self'",
                (req, res) => `'nonce-${res.locals.nonce}'`
            ],
            styleSrc: ["'self'", "'unsafe-inline'"],
            imgSrc: ["'self'", "data:", "https:"],
            fontSrc: ["'self'", "https://fonts.gstatic.com"],
            connectSrc: ["'self'", "https://api.example.com"],
            frameAncestors: ["'none'"],
            formAction: ["'self'"],
            baseUri: ["'self'"],
            objectSrc: ["'none'"],
            upgradeInsecureRequests: []
        }
    })
);

// Использование nonce в шаблоне
app.get('/', (req, res) => {
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <script nonce="${res.locals.nonce}">
                console.log('Этот скрипт разрешён');
            </script>
        </head>
        <body>
            <h1>Страница с CSP</h1>
        </body>
        </html>
    `);
});

app.listen(3000);
```

### Настройка в Python (Flask)

```python
from flask import Flask, make_response
import secrets

app = Flask(__name__)

@app.after_request
def add_security_headers(response):
    # Генерация nonce
    nonce = secrets.token_urlsafe(16)

    csp_policy = "; ".join([
        "default-src 'self'",
        f"script-src 'self' 'nonce-{nonce}'",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "font-src 'self' https://fonts.gstatic.com",
        "connect-src 'self' https://api.example.com",
        "frame-ancestors 'none'",
        "form-action 'self'",
        "base-uri 'self'",
        "object-src 'none'"
    ])

    response.headers['Content-Security-Policy'] = csp_policy

    # Сохраняем nonce для шаблона
    response.headers['X-Nonce'] = nonce

    return response

@app.route('/')
def index():
    return '''
        <!DOCTYPE html>
        <html>
        <head>
            <script nonce="{{nonce}}">
                console.log('Safe script');
            </script>
        </head>
        <body>
            <h1>Защищённая страница</h1>
        </body>
        </html>
    '''

# Использование Flask-Talisman для упрощения
from flask_talisman import Talisman

talisman = Talisman(app, content_security_policy={
    'default-src': "'self'",
    'script-src': ["'self'", "'strict-dynamic'"],
    'style-src': ["'self'", "'unsafe-inline'"],
    'img-src': ["'self'", "data:", "https:"]
})
```

### Настройка в Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # CSP заголовок
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' https://cdn.example.com;
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
        font-src 'self' https://fonts.gstatic.com;
        connect-src 'self' https://api.example.com;
        frame-ancestors 'none';
        form-action 'self';
        base-uri 'self';
        object-src 'none';
        upgrade-insecure-requests;
    " always;

    # Другие заголовки безопасности
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

## Использование nonce и hash

### Nonce (Number used once)

Nonce — уникальный токен, генерируемый для каждого запроса:

```html
<!-- Сервер генерирует nonce и добавляет в заголовок -->
Content-Security-Policy: script-src 'nonce-abc123def456'

<!-- HTML использует тот же nonce -->
<script nonce="abc123def456">
    // Этот скрипт будет выполнен
    console.log('Authorized script');
</script>

<script>
    // Этот скрипт будет заблокирован (нет nonce)
    console.log('Blocked script');
</script>
```

### Hash

Hash позволяет разрешить конкретный inline-скрипт по его хэшу:

```bash
# Генерация SHA-256 хэша
echo -n "console.log('Hello');" | openssl dgst -sha256 -binary | base64
# Результат: LCa0a2j/xo/5m0U8HTBBNBNCLXBkg7+g+YpeiGJm564=
```

```
Content-Security-Policy: script-src 'sha256-LCa0a2j/xo/5m0U8HTBBNBNCLXBkg7+g+YpeiGJm564='
```

```html
<script>console.log('Hello');</script>  <!-- Разрешено -->
<script>console.log('World');</script>  <!-- Заблокировано (другой хэш) -->
```

## Report-Only режим

Для тестирования политики без блокировки:

```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

### Обработка отчётов

```javascript
// Express.js endpoint для CSP-отчётов
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
    const report = req.body['csp-report'];

    console.log('CSP Violation:', {
        documentUri: report['document-uri'],
        violatedDirective: report['violated-directive'],
        blockedUri: report['blocked-uri'],
        sourceFile: report['source-file'],
        lineNumber: report['line-number']
    });

    // Сохраняем в базу или отправляем в систему мониторинга

    res.status(204).send();
});
```

### Формат отчёта

```json
{
    "csp-report": {
        "document-uri": "https://example.com/page",
        "referrer": "",
        "violated-directive": "script-src 'self'",
        "effective-directive": "script-src",
        "original-policy": "default-src 'self'; script-src 'self'",
        "blocked-uri": "https://evil.com/script.js",
        "status-code": 200,
        "source-file": "https://example.com/page",
        "line-number": 15,
        "column-number": 10
    }
}
```

## Strict CSP (Современный подход)

Google рекомендует использовать "Strict CSP" с nonce и strict-dynamic:

```
Content-Security-Policy:
    script-src 'nonce-RANDOM_NONCE' 'strict-dynamic';
    object-src 'none';
    base-uri 'none';
```

### Преимущества Strict CSP

1. **strict-dynamic** — скрипты, загруженные доверенным скриптом, также доверенны
2. **Не нужно перечислять все CDN** — достаточно nonce
3. **Защита от обхода через JSONP**

```html
<!-- С nonce главный скрипт доверенный -->
<script nonce="abc123">
    // Этот скрипт может динамически загружать другие скрипты
    const script = document.createElement('script');
    script.src = 'https://cdn.example.com/lib.js';
    document.head.appendChild(script);  // Разрешено благодаря strict-dynamic
</script>
```

## Типичные ошибки

### 1. Использование 'unsafe-inline' и 'unsafe-eval'

```
# ПЛОХО — отключает защиту от XSS
Content-Security-Policy: script-src 'self' 'unsafe-inline' 'unsafe-eval'

# ХОРОШО — используйте nonce
Content-Security-Policy: script-src 'self' 'nonce-abc123'
```

### 2. Слишком широкие wildcard

```
# ПЛОХО — разрешает любой источник
Content-Security-Policy: script-src *

# ПЛОХО — поддомены могут быть скомпрометированы
Content-Security-Policy: script-src *.example.com

# ХОРОШО — конкретные источники
Content-Security-Policy: script-src 'self' https://cdn.trusted.com
```

### 3. Отсутствие default-src

```
# ПЛОХО — без default-src всё разрешено по умолчанию
Content-Security-Policy: script-src 'self'

# ХОРОШО — явный default-src
Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self'
```

### 4. Не используется object-src 'none'

```
# ПЛОХО — плагины могут использоваться для атак
Content-Security-Policy: default-src 'self'

# ХОРОШО — явно запрещаем плагины
Content-Security-Policy: default-src 'self'; object-src 'none'
```

## Совместимость с фреймворками

### React

```javascript
// Для React нужно использовать nonce для inline-стилей
// webpack.config.js
module.exports = {
    output: {
        // Nonce для webpack chunks
        crossOriginLoading: 'anonymous'
    }
};

// В HTML template
<script nonce="<%= nonce %>">
    window.__webpack_nonce__ = '<%= nonce %>';
</script>
```

### Vue.js

```javascript
// vue.config.js
module.exports = {
    chainWebpack: config => {
        config.plugin('html').tap(args => {
            args[0].cspNonce = '<%= nonce %>';
            return args;
        });
    }
};
```

## Чек-лист CSP

- [ ] Установлен `default-src 'none'` или `default-src 'self'`
- [ ] `script-src` использует nonce или hash, не 'unsafe-inline'
- [ ] `object-src 'none'` для блокировки плагинов
- [ ] `base-uri 'self'` или `'none'`
- [ ] `frame-ancestors` настроен (вместо X-Frame-Options)
- [ ] Тестирование в Report-Only режиме
- [ ] Мониторинг отчётов о нарушениях

## Заключение

CSP — мощный инструмент защиты от XSS и инъекций кода. Ключевые рекомендации:

1. **Начните с Report-Only** — тестируйте политику без блокировки
2. **Используйте nonce** вместо 'unsafe-inline'
3. **Избегайте 'unsafe-eval'** — рефакторьте код, использующий eval()
4. **Минимизируйте whitelist** — меньше источников = меньше риск
5. **Мониторьте нарушения** — настройте report-uri
6. **Постепенно ужесточайте** — не ломайте сайт резкими изменениями
