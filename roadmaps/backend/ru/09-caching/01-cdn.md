# CDN (Content Delivery Network)

[prev: 12-api-lifecycle-management](../08-apis/12-api-lifecycle-management.md) | [next: 02-server-side](./02-server-side.md)

---

## Что такое CDN?

**CDN (Content Delivery Network)** — это географически распределённая сеть серверов, которая доставляет контент пользователям с ближайшего к ним сервера. Это значительно уменьшает задержку (latency) и ускоряет загрузку веб-страниц.

## Как работает CDN?

```
                    ┌─────────────────┐
                    │  Origin Server  │
                    │  (Ваш сервер)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   CDN Network   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐   ┌────────▼────────┐   ┌───────▼───────┐
│  Edge Server  │   │   Edge Server   │   │  Edge Server  │
│   (Москва)    │   │   (Берлин)      │   │  (Токио)      │
└───────┬───────┘   └────────┬────────┘   └───────┬───────┘
        │                    │                    │
   Пользователь         Пользователь         Пользователь
    из России          из Германии           из Японии
```

### Принцип работы:

1. **Первый запрос**: Пользователь запрашивает контент, CDN проверяет кэш на ближайшем edge-сервере
2. **Cache Miss**: Если контента нет, запрос идёт на origin-сервер
3. **Кэширование**: Полученный контент сохраняется на edge-сервере
4. **Последующие запросы**: Контент отдаётся из кэша edge-сервера

## Типы контента для CDN

### Статический контент (основное применение)
- Изображения (JPEG, PNG, WebP, SVG)
- CSS и JavaScript файлы
- Шрифты (WOFF, WOFF2, TTF)
- Видео и аудио файлы
- PDF и другие документы

### Динамический контент
- API-ответы (с коротким TTL)
- Персонализированный контент
- Результаты поисковых запросов

## Настройка CDN

### Конфигурация заголовков кэширования

```python
# Python/Flask - настройка заголовков для статических файлов
from flask import Flask, send_from_directory, make_response

app = Flask(__name__)

@app.route('/static/<path:filename>')
def serve_static(filename):
    response = make_response(send_from_directory('static', filename))

    # Кэшировать на CDN и в браузере на 1 год
    response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'

    # ETag для валидации
    response.headers['ETag'] = generate_etag(filename)

    return response

@app.route('/api/data')
def api_data():
    response = make_response(get_data())

    # Для API: короткий кэш, обязательная ревалидация
    response.headers['Cache-Control'] = 'public, max-age=60, must-revalidate'
    response.headers['Vary'] = 'Accept-Encoding, Authorization'

    return response
```

### Nginx конфигурация для CDN

```nginx
server {
    listen 80;
    server_name example.com;

    # Статические файлы с длинным кэшированием
    location /static/ {
        root /var/www/html;

        # Заголовки для CDN
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header X-Content-Type-Options "nosniff";

        # Включаем gzip для текстовых файлов
        gzip on;
        gzip_types text/css application/javascript application/json;

        # ETag для валидации
        etag on;
    }

    # Изображения с оптимизацией
    location ~* \.(jpg|jpeg|png|gif|webp|svg|ico)$ {
        root /var/www/html;

        add_header Cache-Control "public, max-age=2592000"; # 30 дней
        add_header Vary "Accept-Encoding";

        # Разрешаем CORS для CDN
        add_header Access-Control-Allow-Origin "*";
    }

    # API с коротким кэшем
    location /api/ {
        proxy_pass http://backend;

        # Кэширование на CDN edge-серверах
        add_header Cache-Control "public, s-maxage=60, max-age=0";
        add_header Surrogate-Control "max-age=3600";
    }
}
```

## Популярные CDN-провайдеры

### Cloudflare

```python
# Пример работы с Cloudflare API для очистки кэша
import requests

def purge_cloudflare_cache(zone_id, api_token, urls=None):
    """Очистка кэша Cloudflare"""
    headers = {
        'Authorization': f'Bearer {api_token}',
        'Content-Type': 'application/json'
    }

    if urls:
        # Очистка конкретных URL
        data = {'files': urls}
    else:
        # Полная очистка
        data = {'purge_everything': True}

    response = requests.post(
        f'https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache',
        headers=headers,
        json=data
    )

    return response.json()

# Использование
purge_cloudflare_cache(
    zone_id='your_zone_id',
    api_token='your_api_token',
    urls=['https://example.com/static/style.css']
)
```

### AWS CloudFront

```python
import boto3

def invalidate_cloudfront_cache(distribution_id, paths):
    """Инвалидация кэша CloudFront"""
    client = boto3.client('cloudfront')

    response = client.create_invalidation(
        DistributionId=distribution_id,
        InvalidationBatch={
            'Paths': {
                'Quantity': len(paths),
                'Items': paths  # Например: ['/images/*', '/css/main.css']
            },
            'CallerReference': str(time.time())
        }
    )

    return response['Invalidation']['Id']

# Инвалидация после деплоя
invalidate_cloudfront_cache(
    distribution_id='E1234567890ABC',
    paths=['/static/*', '/index.html']
)
```

## Cache-Control заголовки для CDN

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache-Control директивы                       │
├─────────────────┬───────────────────────────────────────────────┤
│ public          │ Можно кэшировать на CDN и в браузере          │
│ private         │ Только браузерный кэш (не CDN!)               │
│ max-age=N       │ Время жизни кэша в секундах                   │
│ s-maxage=N      │ TTL только для shared cache (CDN)             │
│ no-cache        │ Кэшировать, но всегда ревалидировать          │
│ no-store        │ Не кэшировать вообще                          │
│ immutable       │ Контент никогда не изменится                  │
│ must-revalidate │ Обязательно проверить после истечения TTL     │
│ stale-while-    │ Отдавать устаревший кэш пока обновляется      │
│ revalidate      │                                               │
└─────────────────┴───────────────────────────────────────────────┘
```

## Стратегии версионирования для CDN

### Content Hash в имени файла

```javascript
// webpack.config.js
module.exports = {
    output: {
        filename: '[name].[contenthash].js',
        chunkFilename: '[name].[contenthash].chunk.js'
    }
};

// Результат: main.a1b2c3d4.js
// При изменении файла хэш меняется, CDN получает новый файл
```

### Query String версионирование

```html
<!-- Менее предпочтительный способ (некоторые CDN игнорируют query string) -->
<link rel="stylesheet" href="/css/style.css?v=1.2.3">
<script src="/js/app.js?v=1.2.3"></script>
```

## Лучшие практики

### 1. Правильные заголовки кэширования

```python
# Статика: длинный TTL + immutable
STATIC_CACHE = 'public, max-age=31536000, immutable'

# HTML: короткий TTL или no-cache
HTML_CACHE = 'public, max-age=0, must-revalidate'

# API: зависит от данных
API_CACHE_SHORT = 'public, s-maxage=60, max-age=0'
API_CACHE_LONG = 'public, s-maxage=3600, max-age=300'
```

### 2. Используйте Content Hash

```python
import hashlib

def get_asset_url(filename):
    """Генерация URL с хэшем контента"""
    with open(f'static/{filename}', 'rb') as f:
        content_hash = hashlib.md5(f.read()).hexdigest()[:8]

    name, ext = filename.rsplit('.', 1)
    return f'/static/{name}.{content_hash}.{ext}'

# /static/style.a1b2c3d4.css
```

### 3. Настройте Vary заголовок

```nginx
# Важно для правильного кэширования разных версий
location /api/ {
    add_header Vary "Accept-Encoding, Accept-Language, Authorization";
}
```

## Типичные ошибки

### 1. Кэширование приватных данных

```python
# НЕПРАВИЛЬНО: персональные данные на CDN
@app.route('/api/profile')
def profile():
    response = make_response(get_user_profile())
    response.headers['Cache-Control'] = 'public, max-age=3600'  # Опасно!
    return response

# ПРАВИЛЬНО: приватный кэш
@app.route('/api/profile')
def profile():
    response = make_response(get_user_profile())
    response.headers['Cache-Control'] = 'private, max-age=300'
    return response
```

### 2. Забыли про инвалидацию

```python
# При обновлении контента нужно инвалидировать кэш
def update_product(product_id, data):
    # Обновляем в БД
    db.products.update(product_id, data)

    # Инвалидируем кэш CDN
    invalidate_cdn_cache([
        f'/api/products/{product_id}',
        '/api/products'  # Список тоже устарел
    ])
```

### 3. Неправильный TTL

```python
# Слишком короткий TTL для статики = нагрузка на origin
Cache-Control: max-age=60  # Плохо для статических файлов

# Слишком длинный TTL для динамики = устаревшие данные
Cache-Control: max-age=86400  # Плохо для API
```

## Метрики и мониторинг CDN

```python
# Ключевые метрики для отслеживания
metrics = {
    'cache_hit_ratio': 'Процент запросов из кэша (цель: >90%)',
    'origin_requests': 'Количество запросов к origin-серверу',
    'bandwidth_saved': 'Сэкономленный трафик',
    'latency_p50': 'Медианное время ответа',
    'latency_p99': '99-й перцентиль времени ответа',
    'error_rate': 'Процент ошибок (4xx, 5xx)'
}
```

## Когда использовать CDN?

| Сценарий | Рекомендация |
|----------|--------------|
| Глобальная аудитория | Обязательно |
| Много статического контента | Обязательно |
| Высокая нагрузка | Рекомендуется |
| DDoS-защита нужна | CDN помогает |
| Только локальные пользователи | Опционально |
| Только API без статики | Зависит от нагрузки |

## Резюме

CDN — это критически важный компонент современной веб-инфраструктуры, который:
- Уменьшает latency за счёт географической близости
- Снижает нагрузку на origin-сервер
- Обеспечивает отказоустойчивость
- Защищает от DDoS-атак
- Экономит bandwidth

Правильная настройка кэширования и инвалидации — ключ к эффективному использованию CDN.

---

[prev: 12-api-lifecycle-management](../08-apis/12-api-lifecycle-management.md) | [next: 02-server-side](./02-server-side.md)
