# Клиентское кэширование (браузерное)

## Что такое клиентское кэширование?

**Клиентское кэширование** — это механизм хранения ресурсов (HTML, CSS, JS, изображения) на стороне клиента (браузера), чтобы избежать повторной загрузки при последующих посещениях страницы.

## Виды клиентского кэширования

```
┌─────────────────────────────────────────────────────────────────┐
│                 Виды браузерного кэширования                    │
├─────────────────────────────────────────────────────────────────┤
│  HTTP Cache (Browser Cache)                                     │
│  └── Управляется заголовками Cache-Control, ETag, Expires       │
│                                                                 │
│  Service Worker Cache                                           │
│  └── Программируемый кэш для PWA и offline-режима               │
│                                                                 │
│  Web Storage                                                    │
│  ├── localStorage — постоянное хранилище                        │
│  └── sessionStorage — на время сессии                           │
│                                                                 │
│  IndexedDB                                                      │
│  └── NoSQL база данных в браузере                               │
│                                                                 │
│  Memory Cache                                                   │
│  └── Кэш в оперативной памяти (автоматический)                  │
└─────────────────────────────────────────────────────────────────┘
```

## HTTP Cache (Browser Cache)

### Cache-Control заголовок

```
Cache-Control: max-age=31536000, immutable

┌─────────────────┬───────────────────────────────────────────────┐
│    Директива    │                  Описание                     │
├─────────────────┼───────────────────────────────────────────────┤
│ max-age=N       │ Кэшировать N секунд                           │
│ no-cache        │ Кэшировать, но ревалидировать каждый запрос   │
│ no-store        │ Не кэшировать вообще (конфиденциальные данные)│
│ private         │ Только браузерный кэш (не прокси/CDN)         │
│ public          │ Можно кэшировать везде                        │
│ must-revalidate │ После истечения TTL обязательно проверить     │
│ immutable       │ Ресурс никогда не изменится                   │
│ stale-while-    │ Отдавать устаревший кэш, пока обновляется     │
│ revalidate=N    │                                               │
└─────────────────┴───────────────────────────────────────────────┘
```

### Настройка на сервере (Python/Flask)

```python
from flask import Flask, send_from_directory, make_response, request
import hashlib
import os

app = Flask(__name__)

@app.route('/static/<path:filename>')
def serve_static(filename):
    """Статические файлы с агрессивным кэшированием"""
    response = make_response(send_from_directory('static', filename))

    # Для файлов с хэшем в имени (style.a1b2c3.css)
    if has_content_hash(filename):
        response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'
    else:
        # Для файлов без хэша — короткий кэш с ревалидацией
        response.headers['Cache-Control'] = 'public, max-age=3600, must-revalidate'

    return response

@app.route('/api/user/profile')
def user_profile():
    """Приватные данные — только браузерный кэш"""
    response = make_response(get_current_user_profile())
    response.headers['Cache-Control'] = 'private, max-age=300'
    return response

@app.route('/api/config')
def public_config():
    """Публичные данные с ревалидацией"""
    response = make_response(get_config())
    response.headers['Cache-Control'] = 'public, max-age=0, must-revalidate'

    # ETag для условных запросов
    etag = generate_etag(get_config())
    response.headers['ETag'] = etag

    # Проверяем If-None-Match
    if request.headers.get('If-None-Match') == etag:
        return '', 304

    return response
```

### ETag и условные запросы

```python
import hashlib

def generate_etag(content) -> str:
    """Генерация ETag из содержимого"""
    if isinstance(content, dict):
        content = json.dumps(content, sort_keys=True)
    return hashlib.md5(content.encode()).hexdigest()

@app.route('/api/products/<int:product_id>')
def get_product(product_id):
    product = db.products.find_one({'id': product_id})

    # Генерируем ETag
    etag = f'"{generate_etag(product)}"'

    # Проверяем условный запрос
    if_none_match = request.headers.get('If-None-Match')
    if if_none_match == etag:
        # Данные не изменились — возвращаем 304
        return '', 304

    response = make_response(jsonify(product))
    response.headers['ETag'] = etag
    response.headers['Cache-Control'] = 'private, max-age=0, must-revalidate'

    return response
```

### Nginx конфигурация

```nginx
server {
    listen 80;
    server_name example.com;

    # Статические файлы с content hash
    location ~* \.(js|css)$ {
        root /var/www/static;

        # Если файл содержит хэш (main.a1b2c3.js)
        if ($uri ~* "\.[a-f0-9]{8}\.(js|css)$") {
            add_header Cache-Control "public, max-age=31536000, immutable";
        }

        # Файлы без хэша
        add_header Cache-Control "public, max-age=3600, must-revalidate";

        # ETag автоматический
        etag on;
    }

    # Изображения
    location ~* \.(jpg|jpeg|png|gif|webp|svg|ico)$ {
        root /var/www/static;
        add_header Cache-Control "public, max-age=2592000"; # 30 дней
        etag on;
    }

    # Шрифты
    location ~* \.(woff|woff2|ttf|otf|eot)$ {
        root /var/www/static;
        add_header Cache-Control "public, max-age=31536000, immutable";

        # CORS для шрифтов
        add_header Access-Control-Allow-Origin "*";
    }

    # HTML — не кэшировать или кэшировать минимально
    location ~* \.html$ {
        root /var/www/html;
        add_header Cache-Control "no-cache, must-revalidate";
        etag on;
    }

    # API
    location /api/ {
        proxy_pass http://backend;
        add_header Cache-Control "private, no-cache";
    }
}
```

## Service Worker Cache

### Регистрация Service Worker

```javascript
// main.js
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then(registration => {
            console.log('SW registered:', registration.scope);
        })
        .catch(error => {
            console.log('SW registration failed:', error);
        });
}
```

### Стратегии кэширования в Service Worker

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const STATIC_ASSETS = [
    '/',
    '/index.html',
    '/css/style.css',
    '/js/app.js',
    '/images/logo.png'
];

// Установка — предзагрузка статики
self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(STATIC_ASSETS))
            .then(() => self.skipWaiting())
    );
});

// Активация — очистка старых кэшей
self.addEventListener('activate', event => {
    event.waitUntil(
        caches.keys().then(cacheNames => {
            return Promise.all(
                cacheNames
                    .filter(name => name !== CACHE_NAME)
                    .map(name => caches.delete(name))
            );
        }).then(() => self.clients.claim())
    );
});

// Стратегия: Cache First (для статики)
self.addEventListener('fetch', event => {
    if (event.request.url.includes('/static/')) {
        event.respondWith(
            caches.match(event.request)
                .then(cached => {
                    if (cached) {
                        return cached;
                    }
                    return fetch(event.request).then(response => {
                        const clone = response.clone();
                        caches.open(CACHE_NAME)
                            .then(cache => cache.put(event.request, clone));
                        return response;
                    });
                })
        );
        return;
    }

    // Стратегия: Network First (для API)
    if (event.request.url.includes('/api/')) {
        event.respondWith(
            fetch(event.request)
                .then(response => {
                    const clone = response.clone();
                    caches.open(CACHE_NAME)
                        .then(cache => cache.put(event.request, clone));
                    return response;
                })
                .catch(() => caches.match(event.request))
        );
        return;
    }

    // Стратегия: Stale While Revalidate (для HTML)
    event.respondWith(
        caches.match(event.request)
            .then(cached => {
                const fetchPromise = fetch(event.request)
                    .then(response => {
                        const clone = response.clone();
                        caches.open(CACHE_NAME)
                            .then(cache => cache.put(event.request, clone));
                        return response;
                    });

                return cached || fetchPromise;
            })
    );
});
```

### Workbox — библиотека для Service Worker

```javascript
// sw.js с использованием Workbox
importScripts('https://storage.googleapis.com/workbox-cdn/releases/6.4.1/workbox-sw.js');

const { registerRoute } = workbox.routing;
const { CacheFirst, NetworkFirst, StaleWhileRevalidate } = workbox.strategies;
const { ExpirationPlugin } = workbox.expiration;
const { CacheableResponsePlugin } = workbox.cacheableResponse;

// Статика — Cache First
registerRoute(
    ({ request }) => request.destination === 'style' ||
                     request.destination === 'script' ||
                     request.destination === 'image',
    new CacheFirst({
        cacheName: 'static-assets',
        plugins: [
            new CacheableResponsePlugin({ statuses: [0, 200] }),
            new ExpirationPlugin({
                maxEntries: 100,
                maxAgeSeconds: 30 * 24 * 60 * 60 // 30 дней
            })
        ]
    })
);

// API — Network First с fallback
registerRoute(
    ({ url }) => url.pathname.startsWith('/api/'),
    new NetworkFirst({
        cacheName: 'api-cache',
        networkTimeoutSeconds: 3,
        plugins: [
            new ExpirationPlugin({
                maxEntries: 50,
                maxAgeSeconds: 5 * 60 // 5 минут
            })
        ]
    })
);

// Страницы — Stale While Revalidate
registerRoute(
    ({ request }) => request.mode === 'navigate',
    new StaleWhileRevalidate({
        cacheName: 'pages',
        plugins: [
            new CacheableResponsePlugin({ statuses: [0, 200] })
        ]
    })
);
```

## Web Storage (localStorage / sessionStorage)

### Базовое использование

```javascript
// localStorage — постоянное хранилище
localStorage.setItem('user_preferences', JSON.stringify({
    theme: 'dark',
    language: 'ru'
}));

const prefs = JSON.parse(localStorage.getItem('user_preferences'));

// sessionStorage — только на время сессии
sessionStorage.setItem('cart', JSON.stringify([
    { id: 1, quantity: 2 },
    { id: 5, quantity: 1 }
]));
```

### Обёртка с TTL

```javascript
class StorageWithTTL {
    constructor(storage = localStorage) {
        this.storage = storage;
    }

    set(key, value, ttlSeconds = 3600) {
        const item = {
            value: value,
            expiresAt: Date.now() + ttlSeconds * 1000
        };
        this.storage.setItem(key, JSON.stringify(item));
    }

    get(key) {
        const itemStr = this.storage.getItem(key);
        if (!itemStr) return null;

        try {
            const item = JSON.parse(itemStr);

            if (Date.now() > item.expiresAt) {
                this.storage.removeItem(key);
                return null;
            }

            return item.value;
        } catch (e) {
            return null;
        }
    }

    remove(key) {
        this.storage.removeItem(key);
    }

    clear() {
        this.storage.clear();
    }
}

// Использование
const cache = new StorageWithTTL();
cache.set('api_response', { data: [1, 2, 3] }, 300); // 5 минут
const data = cache.get('api_response');
```

### Кэширование API-ответов

```javascript
class APICache {
    constructor(options = {}) {
        this.storage = new StorageWithTTL();
        this.defaultTTL = options.defaultTTL || 300;
    }

    async fetch(url, options = {}) {
        const cacheKey = this.getCacheKey(url, options);
        const ttl = options.cacheTTL || this.defaultTTL;

        // Проверяем кэш
        if (!options.forceRefresh) {
            const cached = this.storage.get(cacheKey);
            if (cached) {
                return cached;
            }
        }

        // Делаем запрос
        const response = await fetch(url, options);
        const data = await response.json();

        // Кэшируем успешные ответы
        if (response.ok) {
            this.storage.set(cacheKey, data, ttl);
        }

        return data;
    }

    getCacheKey(url, options) {
        const params = options.body ? JSON.stringify(options.body) : '';
        return `api:${url}:${params}`;
    }

    invalidate(urlPattern) {
        // Удаляем все ключи, matching паттерну
        for (let i = 0; i < localStorage.length; i++) {
            const key = localStorage.key(i);
            if (key && key.includes(urlPattern)) {
                localStorage.removeItem(key);
            }
        }
    }
}

// Использование
const api = new APICache({ defaultTTL: 600 });

// Кэшированный запрос
const products = await api.fetch('/api/products', { cacheTTL: 3600 });

// Принудительное обновление
const fresh = await api.fetch('/api/products', { forceRefresh: true });
```

## IndexedDB для больших данных

```javascript
class IndexedDBCache {
    constructor(dbName = 'app-cache', version = 1) {
        this.dbName = dbName;
        this.version = version;
        this.db = null;
    }

    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.version);

            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                this.db = request.result;
                resolve(this);
            };

            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                if (!db.objectStoreNames.contains('cache')) {
                    const store = db.createObjectStore('cache', { keyPath: 'key' });
                    store.createIndex('expiresAt', 'expiresAt', { unique: false });
                }
            };
        });
    }

    async set(key, value, ttlSeconds = 3600) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['cache'], 'readwrite');
            const store = transaction.objectStore('cache');

            const item = {
                key: key,
                value: value,
                expiresAt: Date.now() + ttlSeconds * 1000,
                createdAt: Date.now()
            };

            const request = store.put(item);
            request.onsuccess = () => resolve();
            request.onerror = () => reject(request.error);
        });
    }

    async get(key) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['cache'], 'readonly');
            const store = transaction.objectStore('cache');

            const request = store.get(key);

            request.onsuccess = () => {
                const item = request.result;

                if (!item) {
                    resolve(null);
                    return;
                }

                if (Date.now() > item.expiresAt) {
                    this.delete(key);
                    resolve(null);
                    return;
                }

                resolve(item.value);
            };

            request.onerror = () => reject(request.error);
        });
    }

    async delete(key) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['cache'], 'readwrite');
            const store = transaction.objectStore('cache');

            const request = store.delete(key);
            request.onsuccess = () => resolve();
            request.onerror = () => reject(request.error);
        });
    }

    async cleanup() {
        // Удаляем просроченные записи
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction(['cache'], 'readwrite');
            const store = transaction.objectStore('cache');
            const index = store.index('expiresAt');

            const range = IDBKeyRange.upperBound(Date.now());
            const request = index.openCursor(range);

            request.onsuccess = (event) => {
                const cursor = event.target.result;
                if (cursor) {
                    store.delete(cursor.primaryKey);
                    cursor.continue();
                } else {
                    resolve();
                }
            };

            request.onerror = () => reject(request.error);
        });
    }
}

// Использование
const cache = new IndexedDBCache();
await cache.init();

// Кэширование больших данных
await cache.set('large_dataset', arrayOf10000Items, 86400);
const data = await cache.get('large_dataset');
```

## Лучшие практики

### 1. Выбор правильного типа кэша

```
┌─────────────────────┬─────────────────────────────────────────┐
│ Тип данных          │ Рекомендуемый кэш                       │
├─────────────────────┼─────────────────────────────────────────┤
│ Статика (CSS, JS)   │ HTTP Cache + Service Worker             │
│ API ответы          │ localStorage с TTL или IndexedDB        │
│ Пользовательские    │ sessionStorage                          │
│ данные сессии       │                                         │
│ Большие данные      │ IndexedDB                               │
│ Offline-режим       │ Service Worker Cache                    │
└─────────────────────┴─────────────────────────────────────────┘
```

### 2. Версионирование статики

```javascript
// В HTML используйте хэши в именах файлов
<link rel="stylesheet" href="/css/style.a1b2c3d4.css">
<script src="/js/app.e5f6g7h8.js"></script>

// Это позволяет:
// 1. Использовать immutable cache (max-age=31536000)
// 2. Автоматически инвалидировать при изменении
```

### 3. Обработка ошибок хранилища

```javascript
function safeLocalStorage(key, value) {
    try {
        if (value === undefined) {
            return localStorage.getItem(key);
        }
        localStorage.setItem(key, value);
        return true;
    } catch (e) {
        // QuotaExceededError или приватный режим
        console.warn('localStorage unavailable:', e);
        return null;
    }
}
```

## Типичные ошибки

### 1. Кэширование HTML с длинным TTL

```
# НЕПРАВИЛЬНО: HTML закэширован надолго
Cache-Control: max-age=86400

# При обновлении приложения пользователи видят старую версию

# ПРАВИЛЬНО:
Cache-Control: no-cache, must-revalidate
# или
Cache-Control: max-age=0, must-revalidate
```

### 2. Забыли про private/public

```python
# НЕПРАВИЛЬНО: приватные данные могут попасть в shared cache
@app.route('/api/my-orders')
def my_orders():
    response = make_response(get_user_orders())
    response.headers['Cache-Control'] = 'max-age=300'  # Опасно!
    return response

# ПРАВИЛЬНО:
response.headers['Cache-Control'] = 'private, max-age=300'
```

### 3. Переполнение localStorage

```javascript
// Проверяем доступное место перед записью
function getStorageUsage() {
    let total = 0;
    for (let key in localStorage) {
        if (localStorage.hasOwnProperty(key)) {
            total += localStorage[key].length * 2; // UTF-16
        }
    }
    return total;
}

// localStorage обычно ограничен 5-10 MB
const MAX_STORAGE = 5 * 1024 * 1024; // 5 MB

if (getStorageUsage() > MAX_STORAGE * 0.9) {
    // Очищаем старые записи
    cleanupOldEntries();
}
```

## Резюме

Клиентское кэширование — критически важная техника для производительности веб-приложений:

- **HTTP Cache** — основа для статических ресурсов
- **Service Worker** — для PWA и offline-режима
- **localStorage/sessionStorage** — для простых данных
- **IndexedDB** — для больших объёмов данных

Правильная настройка заголовков кэширования и выбор подходящего типа хранилища напрямую влияют на UX и скорость загрузки приложения.
