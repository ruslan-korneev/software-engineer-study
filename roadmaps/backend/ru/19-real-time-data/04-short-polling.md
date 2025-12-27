# Short Polling (Короткий опрос)

## Введение

**Short Polling** (короткий опрос) — это самый простой и базовый метод получения обновлений от сервера в реальном времени. Суть метода заключается в том, что клиент периодически отправляет HTTP-запросы к серверу через фиксированные интервалы времени, чтобы проверить, появились ли новые данные.

Этот подход существует с самых ранних дней веб-разработки и до сих пор используется в некоторых сценариях благодаря своей простоте реализации и универсальности.

### Ключевые характеристики

- **Простота**: Не требует специальных протоколов или технологий
- **Универсальность**: Работает с любым HTTP-сервером
- **Предсказуемость**: Фиксированные интервалы запросов
- **Stateless**: Каждый запрос независим от предыдущих

### Историческая справка

Short Polling был первым методом создания "real-time" веб-приложений ещё до появления WebSocket, Server-Sent Events и других современных технологий. В эпоху Web 1.0 это был единственный способ обновлять данные на странице без полной её перезагрузки.

---

## Как работает Short Polling

### Механизм работы

Short Polling работает по следующему принципу:

1. Клиент устанавливает таймер с определённым интервалом (например, каждые 5 секунд)
2. По истечении интервала клиент отправляет HTTP-запрос к серверу
3. Сервер немедленно отвечает текущим состоянием данных (или пустым ответом)
4. Клиент обрабатывает ответ и обновляет интерфейс при необходимости
5. Цикл повторяется

### ASCII-диаграмма

```
                    Short Polling: Временная диаграмма
                    ==================================

Клиент                                                    Сервер
   │                                                         │
   │  ─────────── Запрос 1 ───────────>                     │
   │                                                         │
   │  <────────── Ответ (нет данных) ──                     │
   │                                                         │
   │         [Ожидание интервала]                           │
   │         ~~~~ 5 секунд ~~~~                             │
   │                                                         │
   │  ─────────── Запрос 2 ───────────>                     │
   │                                                         │
   │  <────────── Ответ (нет данных) ──                     │
   │                                                         │
   │         [Ожидание интервала]                           │
   │         ~~~~ 5 секунд ~~~~                             │
   │                                                         │
   │  ─────────── Запрос 3 ───────────>                     │
   │                                                         │  Появились
   │  <────────── Ответ (ЕСТЬ ДАННЫЕ!) ─                    │  новые данные!
   │                                                         │
   │         [Обновление UI]                                │
   │         [Ожидание интервала]                           │
   │         ~~~~ 5 секунд ~~~~                             │
   │                                                         │
   │  ─────────── Запрос 4 ───────────>                     │
   │                                                         │
   │  <────────── Ответ (нет данных) ──                     │
   │                                                         │
   ▼                                                         ▼
```

### Детальная схема HTTP-обмена

```
┌─────────────────────────────────────────────────────────────────┐
│                     SHORT POLLING FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐                              ┌──────────┐        │
│  │  Client  │                              │  Server  │        │
│  └────┬─────┘                              └────┬─────┘        │
│       │                                         │               │
│       │  setInterval(() => fetch(), 5000)      │               │
│       │  ◄────────────────────────────────     │               │
│       │                                         │               │
│       │                                         │               │
│  t=0  │ ──── GET /api/updates ──────────────► │               │
│       │                                         │ Проверка БД  │
│       │ ◄─── 200 OK { updates: [] } ────────── │               │
│       │                                         │               │
│       │         ~~ ожидание 5 сек ~~           │               │
│       │                                         │               │
│  t=5  │ ──── GET /api/updates ──────────────► │               │
│       │                                         │ Проверка БД  │
│       │ ◄─── 200 OK { updates: [] } ────────── │               │
│       │                                         │               │
│       │         ~~ ожидание 5 сек ~~           │               │
│       │                                         │               │
│  t=10 │ ──── GET /api/updates ──────────────► │               │
│       │                                         │ Проверка БД  │
│       │ ◄─── 200 OK { updates: [...] } ─────── │ ← Новые!     │
│       │                                         │               │
│       │  [Обновление интерфейса]               │               │
│       │                                         │               │
│       ▼                                         ▼               │
└─────────────────────────────────────────────────────────────────┘
```

### Жизненный цикл запроса

```
┌─────────────────────────────────────────────────────────────┐
│                    Один цикл Short Polling                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. СОЗДАНИЕ ЗАПРОСА                                        │
│     ┌────────────────────────────────────┐                  │
│     │ const response = await fetch(url); │                  │
│     └────────────────────────────────────┘                  │
│                        │                                    │
│                        ▼                                    │
│  2. ОТПРАВКА НА СЕРВЕР                                      │
│     ┌────────────────────────────────────┐                  │
│     │ GET /api/updates HTTP/1.1          │                  │
│     │ Host: example.com                  │                  │
│     │ Accept: application/json           │                  │
│     └────────────────────────────────────┘                  │
│                        │                                    │
│                        ▼                                    │
│  3. ОБРАБОТКА НА СЕРВЕРЕ (мгновенная)                       │
│     ┌────────────────────────────────────┐                  │
│     │ - Проверка базы данных             │                  │
│     │ - Формирование ответа              │                  │
│     │ - Отправка клиенту                 │                  │
│     └────────────────────────────────────┘                  │
│                        │                                    │
│                        ▼                                    │
│  4. ПОЛУЧЕНИЕ ОТВЕТА                                        │
│     ┌────────────────────────────────────┐                  │
│     │ HTTP/1.1 200 OK                    │                  │
│     │ Content-Type: application/json     │                  │
│     │                                    │                  │
│     │ {"updates": [], "timestamp": ...}  │                  │
│     └────────────────────────────────────┘                  │
│                        │                                    │
│                        ▼                                    │
│  5. ОЖИДАНИЕ ИНТЕРВАЛА                                      │
│     ┌────────────────────────────────────┐                  │
│     │ setTimeout(() => poll(), 5000);    │                  │
│     └────────────────────────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Примеры кода

### JavaScript клиент (Браузер)

#### Базовая реализация с setInterval

```javascript
/**
 * Простейший пример Short Polling
 * Опрос сервера каждые 5 секунд
 */
class SimpleShortPolling {
    constructor(url, intervalMs = 5000) {
        this.url = url;
        this.intervalMs = intervalMs;
        this.intervalId = null;
        this.isPolling = false;
    }

    // Запуск polling
    start() {
        if (this.isPolling) {
            console.warn('Polling уже запущен');
            return;
        }

        this.isPolling = true;
        console.log(`Запуск Short Polling с интервалом ${this.intervalMs}ms`);

        // Первый запрос сразу
        this.poll();

        // Последующие запросы по интервалу
        this.intervalId = setInterval(() => this.poll(), this.intervalMs);
    }

    // Остановка polling
    stop() {
        if (this.intervalId) {
            clearInterval(this.intervalId);
            this.intervalId = null;
        }
        this.isPolling = false;
        console.log('Short Polling остановлен');
    }

    // Выполнение одного запроса
    async poll() {
        try {
            const response = await fetch(this.url);

            if (!response.ok) {
                throw new Error(`HTTP ошибка: ${response.status}`);
            }

            const data = await response.json();
            this.handleData(data);

        } catch (error) {
            console.error('Ошибка polling:', error);
            this.handleError(error);
        }
    }

    // Обработка полученных данных
    handleData(data) {
        if (data.updates && data.updates.length > 0) {
            console.log('Получены обновления:', data.updates);
            // Здесь логика обновления UI
        }
    }

    // Обработка ошибок
    handleError(error) {
        // Можно реализовать логику повторных попыток
        // или увеличение интервала при ошибках
    }
}

// Использование
const polling = new SimpleShortPolling('/api/updates', 5000);
polling.start();

// Для остановки
// polling.stop();
```

#### Продвинутая реализация с обработкой ошибок

```javascript
/**
 * Продвинутый Short Polling с:
 * - Экспоненциальным backoff при ошибках
 * - Отслеживанием последнего timestamp
 * - Отменой запросов через AbortController
 * - Событийной моделью
 */
class AdvancedShortPolling extends EventTarget {
    constructor(options = {}) {
        super();

        this.url = options.url || '/api/updates';
        this.intervalMs = options.intervalMs || 5000;
        this.maxRetries = options.maxRetries || 5;
        this.backoffMultiplier = options.backoffMultiplier || 2;

        this.intervalId = null;
        this.abortController = null;
        this.isPolling = false;
        this.retryCount = 0;
        this.lastTimestamp = null;
    }

    start() {
        if (this.isPolling) return;

        this.isPolling = true;
        this.retryCount = 0;

        this.emit('start');
        this.schedulePoll(0); // Первый запрос немедленно
    }

    stop() {
        this.isPolling = false;

        if (this.intervalId) {
            clearTimeout(this.intervalId);
            this.intervalId = null;
        }

        if (this.abortController) {
            this.abortController.abort();
            this.abortController = null;
        }

        this.emit('stop');
    }

    schedulePoll(delay) {
        if (!this.isPolling) return;

        this.intervalId = setTimeout(() => this.poll(), delay);
    }

    async poll() {
        if (!this.isPolling) return;

        this.abortController = new AbortController();
        const signal = this.abortController.signal;

        try {
            // Формируем URL с параметрами
            const url = new URL(this.url, window.location.origin);
            if (this.lastTimestamp) {
                url.searchParams.set('since', this.lastTimestamp);
            }

            const response = await fetch(url, {
                method: 'GET',
                headers: {
                    'Accept': 'application/json',
                    'X-Requested-With': 'XMLHttpRequest'
                },
                signal
            });

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            const data = await response.json();

            // Успешный запрос — сброс счётчика ошибок
            this.retryCount = 0;

            // Обновляем timestamp
            if (data.timestamp) {
                this.lastTimestamp = data.timestamp;
            }

            // Отправляем событие с данными
            if (data.updates && data.updates.length > 0) {
                this.emit('data', data.updates);
            }

            // Планируем следующий запрос
            this.schedulePoll(this.intervalMs);

        } catch (error) {
            if (error.name === 'AbortError') {
                // Запрос был отменён — это нормально
                return;
            }

            this.emit('error', error);
            this.handleRetry();
        }
    }

    handleRetry() {
        if (!this.isPolling) return;

        this.retryCount++;

        if (this.retryCount > this.maxRetries) {
            this.emit('maxRetriesExceeded');
            this.stop();
            return;
        }

        // Экспоненциальный backoff
        const delay = this.intervalMs * Math.pow(this.backoffMultiplier, this.retryCount);
        console.log(`Повторная попытка ${this.retryCount}/${this.maxRetries} через ${delay}ms`);

        this.schedulePoll(delay);
    }

    emit(type, detail = null) {
        this.dispatchEvent(new CustomEvent(type, { detail }));
    }
}

// Использование с обработкой событий
const polling = new AdvancedShortPolling({
    url: '/api/notifications',
    intervalMs: 3000,
    maxRetries: 3
});

polling.addEventListener('data', (event) => {
    console.log('Новые уведомления:', event.detail);
    updateNotificationBadge(event.detail.length);
});

polling.addEventListener('error', (event) => {
    console.error('Ошибка:', event.detail);
});

polling.addEventListener('maxRetriesExceeded', () => {
    showConnectionError();
});

polling.start();
```

#### Пример для чата с визуальным обновлением

```javascript
/**
 * Short Polling для простого чата
 */
class ChatPolling {
    constructor(chatContainerId) {
        this.container = document.getElementById(chatContainerId);
        this.lastMessageId = 0;
        this.polling = null;
    }

    init() {
        this.polling = new AdvancedShortPolling({
            url: '/api/chat/messages',
            intervalMs: 2000
        });

        this.polling.addEventListener('data', (event) => {
            this.renderMessages(event.detail);
        });

        this.polling.start();
    }

    renderMessages(messages) {
        messages.forEach(msg => {
            if (msg.id > this.lastMessageId) {
                const messageEl = document.createElement('div');
                messageEl.className = `message ${msg.isOwn ? 'own' : 'other'}`;
                messageEl.innerHTML = `
                    <span class="author">${msg.author}</span>
                    <p class="text">${msg.text}</p>
                    <span class="time">${msg.time}</span>
                `;
                this.container.appendChild(messageEl);
                this.lastMessageId = msg.id;
            }
        });

        // Прокрутка вниз
        this.container.scrollTop = this.container.scrollHeight;
    }

    destroy() {
        if (this.polling) {
            this.polling.stop();
        }
    }
}
```

---

### Python сервер (Flask)

#### Базовый сервер на Flask

```python
"""
Short Polling сервер на Flask
Простой пример с in-memory хранилищем
"""

from flask import Flask, jsonify, request
from datetime import datetime
import threading
import time

app = Flask(__name__)

# In-memory хранилище обновлений
updates_store = {
    'updates': [],
    'last_id': 0,
    'lock': threading.Lock()
}


@app.route('/api/updates', methods=['GET'])
def get_updates():
    """
    Endpoint для Short Polling
    Возвращает обновления, появившиеся после указанного timestamp
    """
    since = request.args.get('since', type=float, default=0)

    with updates_store['lock']:
        # Фильтруем обновления по времени
        new_updates = [
            update for update in updates_store['updates']
            if update['timestamp'] > since
        ]

    return jsonify({
        'updates': new_updates,
        'timestamp': datetime.now().timestamp(),
        'count': len(new_updates)
    })


@app.route('/api/updates', methods=['POST'])
def add_update():
    """
    Добавление нового обновления
    (для тестирования и демонстрации)
    """
    data = request.get_json()

    with updates_store['lock']:
        updates_store['last_id'] += 1
        update = {
            'id': updates_store['last_id'],
            'message': data.get('message', ''),
            'timestamp': datetime.now().timestamp(),
            'created_at': datetime.now().isoformat()
        }
        updates_store['updates'].append(update)

        # Храним только последние 100 обновлений
        if len(updates_store['updates']) > 100:
            updates_store['updates'] = updates_store['updates'][-100:]

    return jsonify(update), 201


# Статистика для мониторинга
request_counter = {'count': 0, 'start_time': time.time()}


@app.before_request
def count_requests():
    """Подсчёт запросов для демонстрации нагрузки"""
    request_counter['count'] += 1


@app.route('/api/stats', methods=['GET'])
def get_stats():
    """Статистика polling-запросов"""
    elapsed = time.time() - request_counter['start_time']
    return jsonify({
        'total_requests': request_counter['count'],
        'uptime_seconds': elapsed,
        'requests_per_second': request_counter['count'] / elapsed if elapsed > 0 else 0
    })


if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

#### Продвинутый сервер на FastAPI

```python
"""
Short Polling сервер на FastAPI
С более продвинутыми возможностями
"""

from fastapi import FastAPI, Query, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime
import asyncio
from collections import deque
import logging

# Настройка логирования
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Short Polling API",
    description="Пример сервера для Short Polling",
    version="1.0.0"
)

# CORS для кросс-доменных запросов
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# Pydantic модели
class UpdateCreate(BaseModel):
    message: str
    priority: Optional[str] = "normal"


class Update(BaseModel):
    id: int
    message: str
    priority: str
    timestamp: float
    created_at: str


class UpdatesResponse(BaseModel):
    updates: List[Update]
    timestamp: float
    count: int
    has_more: bool


# In-memory хранилище (в продакшене используйте Redis/БД)
class UpdatesStore:
    def __init__(self, max_size: int = 1000):
        self.updates: deque = deque(maxlen=max_size)
        self.last_id: int = 0
        self._lock = asyncio.Lock()

    async def add(self, message: str, priority: str = "normal") -> Update:
        async with self._lock:
            self.last_id += 1
            now = datetime.now()
            update = Update(
                id=self.last_id,
                message=message,
                priority=priority,
                timestamp=now.timestamp(),
                created_at=now.isoformat()
            )
            self.updates.append(update)
            logger.info(f"Добавлено обновление #{self.last_id}: {message[:50]}...")
            return update

    async def get_since(
        self,
        since: float,
        limit: int = 50,
        priority: Optional[str] = None
    ) -> tuple[List[Update], bool]:
        async with self._lock:
            filtered = [
                u for u in self.updates
                if u.timestamp > since and (priority is None or u.priority == priority)
            ]
            has_more = len(filtered) > limit
            return filtered[:limit], has_more


store = UpdatesStore()


# Метрики
class Metrics:
    def __init__(self):
        self.request_count = 0
        self.start_time = datetime.now()
        self.empty_responses = 0
        self.data_responses = 0

    def record_request(self, has_data: bool):
        self.request_count += 1
        if has_data:
            self.data_responses += 1
        else:
            self.empty_responses += 1


metrics = Metrics()


@app.get("/api/updates", response_model=UpdatesResponse)
async def get_updates(
    since: float = Query(0, description="Timestamp для фильтрации"),
    limit: int = Query(50, ge=1, le=100, description="Максимум обновлений"),
    priority: Optional[str] = Query(None, description="Фильтр по приоритету")
):
    """
    Получение обновлений (Short Polling endpoint)

    - **since**: Вернуть обновления после этого timestamp
    - **limit**: Максимальное количество обновлений
    - **priority**: Опциональный фильтр по приоритету (low/normal/high)
    """
    updates, has_more = await store.get_since(since, limit, priority)

    has_data = len(updates) > 0
    metrics.record_request(has_data)

    return UpdatesResponse(
        updates=updates,
        timestamp=datetime.now().timestamp(),
        count=len(updates),
        has_more=has_more
    )


@app.post("/api/updates", response_model=Update, status_code=201)
async def create_update(update_data: UpdateCreate):
    """Создание нового обновления"""
    if not update_data.message.strip():
        raise HTTPException(status_code=400, detail="Сообщение не может быть пустым")

    update = await store.add(
        message=update_data.message,
        priority=update_data.priority
    )
    return update


@app.get("/api/stats")
async def get_stats():
    """Статистика сервера"""
    elapsed = (datetime.now() - metrics.start_time).total_seconds()

    # Расчёт эффективности: % запросов с реальными данными
    efficiency = 0
    if metrics.request_count > 0:
        efficiency = (metrics.data_responses / metrics.request_count) * 100

    return {
        "total_requests": metrics.request_count,
        "requests_with_data": metrics.data_responses,
        "empty_requests": metrics.empty_responses,
        "efficiency_percent": round(efficiency, 2),
        "uptime_seconds": round(elapsed, 2),
        "requests_per_second": round(metrics.request_count / elapsed, 2) if elapsed > 0 else 0,
        "updates_in_store": len(store.updates)
    }


# Генератор тестовых данных
async def generate_test_updates():
    """Фоновая задача для генерации тестовых обновлений"""
    messages = [
        "Новый пользователь зарегистрировался",
        "Товар добавлен в корзину",
        "Заказ оформлен",
        "Оплата получена",
        "Комментарий добавлен",
    ]
    import random

    while True:
        await asyncio.sleep(random.randint(3, 10))
        message = random.choice(messages)
        priority = random.choice(["low", "normal", "high"])
        await store.add(message, priority)


@app.on_event("startup")
async def startup():
    """Запуск фоновых задач"""
    # Можно раскомментировать для автогенерации тестовых данных
    # asyncio.create_task(generate_test_updates())
    logger.info("Short Polling сервер запущен")


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Преимущества и недостатки

### Сравнительная таблица

| Аспект | Преимущества | Недостатки |
|--------|-------------|------------|
| **Простота реализации** | Минимальный код на клиенте и сервере. Обычный HTTP, без специальных библиотек | - |
| **Совместимость** | Работает везде: любой браузер, любой HTTP-сервер, через прокси и файрволы | - |
| **Масштабирование** | Легко масштабируется горизонтально (stateless) | - |
| **Отладка** | Легко отслеживать в DevTools как обычные HTTP-запросы | - |
| **Нагрузка на сервер** | - | Высокая: множество "пустых" запросов |
| **Задержка доставки** | - | До размера интервала (например, 5 сек) |
| **Трафик** | - | Избыточный: заголовки HTTP при каждом запросе |
| **Батарея мобильных** | - | Повышенный расход из-за частых запросов |
| **Real-time** | - | Не истинный real-time, а псевдо-реальное время |

### Детальный анализ преимуществ

#### 1. Простота реализации

```javascript
// Минимальная реализация — всего несколько строк
setInterval(async () => {
    const data = await fetch('/api/updates').then(r => r.json());
    if (data.updates.length) updateUI(data.updates);
}, 5000);
```

#### 2. Универсальная совместимость

- Работает с HTTP/1.0, HTTP/1.1, HTTP/2, HTTP/3
- Не требует WebSocket-совместимого сервера
- Проходит через корпоративные прокси и файрволы
- Работает в старых браузерах (IE6+)

#### 3. Stateless архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                  Stateless преимущества                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐      ┌───────────────┐      ┌─────────┐       │
│  │ Client  │──────│ Load Balancer │──────│ Server1 │       │
│  │         │      │               │      ├─────────┤       │
│  │         │      │   (любой      │      │ Server2 │       │
│  │         │      │   сервер)     │      ├─────────┤       │
│  │         │      │               │      │ Server3 │       │
│  └─────────┘      └───────────────┘      └─────────┘       │
│                                                             │
│  Каждый запрос может обрабатываться разным сервером!       │
│  Нет привязки к конкретному инстансу.                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Детальный анализ недостатков

#### 1. Высокая нагрузка на сервер

```
Пример: 10,000 активных пользователей, интервал 5 секунд

Запросов в секунду = 10,000 / 5 = 2,000 req/s

При этом 95% запросов возвращают пустой ответ!
Эффективных запросов: ~100 req/s
Потери: 1,900 req/s "впустую"
```

#### 2. Задержка доставки данных

```
┌─────────────────────────────────────────────────────────────┐
│              Худший случай задержки                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Время: 0s     5s     10s    15s    20s                     │
│         │      │      │      │      │                       │
│         ▼      ▼      ▼      ▼      ▼                       │
│  Poll:  ●──────●──────●──────●──────●                       │
│                ▲                                            │
│                │                                            │
│              Данные появились здесь (t=5.1s)                │
│              Клиент узнает только в t=10s                   │
│              Задержка: 4.9 секунды!                         │
│                                                             │
│  Средняя задержка = interval / 2 = 2.5 секунды              │
│  Максимальная задержка = interval - ε ≈ 5 секунд            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3. Избыточный трафик

```
Размер одного polling-запроса:

Request Headers:  ~400-800 bytes
Response Headers: ~200-400 bytes
Empty Body:       ~50 bytes
─────────────────────────────
Итого:            ~650-1250 bytes

При 10,000 пользователей и 5 сек интервале:
2,000 req/s × 1000 bytes = 2 MB/s только на пустые ответы!
За час: 7.2 GB трафика "впустую"
```

---

## Когда использовать Short Polling

### Идеальные сценарии

```
┌─────────────────────────────────────────────────────────────┐
│              КОГДА SHORT POLLING — ХОРОШИЙ ВЫБОР            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✓ Простые админ-панели с небольшим числом пользователей   │
│  ✓ Мониторинг с редкими обновлениями (раз в минуту+)       │
│  ✓ MVP/прототипы, где важна скорость разработки            │
│  ✓ Legacy-системы без поддержки WebSocket                  │
│  ✓ Среды с ограничениями (строгие прокси, файрволы)        │
│  ✓ Данные обновляются предсказуемо (каждые N минут)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Практические примеры использования

#### 1. Статус сборки CI/CD

```javascript
// Проверка статуса каждые 30 секунд — редкие обновления
class BuildStatusPolling {
    constructor(buildId) {
        this.buildId = buildId;
        this.intervalMs = 30000; // 30 секунд достаточно
    }

    async poll() {
        const status = await fetch(`/api/builds/${this.buildId}/status`)
            .then(r => r.json());

        this.updateUI(status);

        // Остановка при завершении сборки
        if (['success', 'failed', 'cancelled'].includes(status.state)) {
            this.stop();
        }
    }
}
```

#### 2. Курсы валют

```javascript
// Курсы обновляются раз в минуту
const currencyPolling = {
    interval: 60000, // 1 минута

    async fetchRates() {
        const rates = await fetch('/api/currency/rates').then(r => r.json());
        document.getElementById('usd-rate').textContent = rates.USD;
        document.getElementById('eur-rate').textContent = rates.EUR;
    }
};

setInterval(() => currencyPolling.fetchRates(), currencyPolling.interval);
```

#### 3. Админ-панель (мало пользователей)

```javascript
// Админов обычно немного — нагрузка не критична
class AdminDashboardPolling {
    constructor() {
        this.metrics = new MetricsPolling('/api/admin/metrics', 10000);
        this.alerts = new AlertsPolling('/api/admin/alerts', 5000);
        this.tasks = new TasksPolling('/api/admin/tasks', 15000);
    }

    start() {
        this.metrics.start();
        this.alerts.start();
        this.tasks.start();
    }
}
```

### Когда НЕ использовать

```
┌─────────────────────────────────────────────────────────────┐
│              КОГДА SHORT POLLING — ПЛОХОЙ ВЫБОР             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✗ Чаты и мессенджеры (нужен мгновенный ответ)             │
│  ✗ Многопользовательские игры (критична задержка)          │
│  ✗ Финансовые приложения (real-time котировки)             │
│  ✗ Коллаборативное редактирование (Google Docs)            │
│  ✗ Высоконагруженные системы (100K+ пользователей)         │
│  ✗ Мобильные приложения (экономия батареи)                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Сравнение с другими методами

### Обзорная таблица

| Характеристика | Short Polling | Long Polling | SSE | WebSocket |
|---------------|---------------|--------------|-----|-----------|
| **Сложность** | Очень низкая | Низкая | Средняя | Высокая |
| **Задержка** | До интервала | Минимальная | Минимальная | Минимальная |
| **Двунаправленность** | Нет | Нет | Нет | Да |
| **Нагрузка на сервер** | Высокая | Средняя | Низкая | Низкая |
| **Масштабируемость** | Хорошая | Средняя | Хорошая | Сложная |
| **Совместимость** | 100% | 100% | 95% | 90% |
| **Держит соединение** | Нет | Частично | Да | Да |

### Визуальное сравнение

```
┌─────────────────────────────────────────────────────────────────────┐
│                        СРАВНЕНИЕ МЕТОДОВ                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SHORT POLLING                                                      │
│  ═══════════════                                                    │
│  Client ──► Server   Client ──► Server   Client ──► Server         │
│         ◄──             ◄──             ◄──                         │
│     [5s]            [5s]            [5s]                            │
│                                                                     │
│  Множество коротких соединений через равные интервалы               │
│                                                                     │
│  ───────────────────────────────────────────────────────────────    │
│                                                                     │
│  LONG POLLING                                                       │
│  ═══════════════                                                    │
│  Client ──────────────────────────────► Server                      │
│         ◄────────────────────────────── (ждёт данные)               │
│  Client ──────────────────────────────► Server                      │
│         ◄────────────────────────────── (ждёт данные)               │
│                                                                     │
│  Запрос висит, пока не появятся данные                              │
│                                                                     │
│  ───────────────────────────────────────────────────────────────    │
│                                                                     │
│  SERVER-SENT EVENTS (SSE)                                           │
│  ═══════════════════════════                                        │
│  Client ─────────────────────────────────────────► Server           │
│         ◄════════════════════════════════════════  (поток)          │
│                    ◄─ event ─┘  ◄─ event ─┘  ◄─ event ─┘            │
│                                                                     │
│  Одно соединение, сервер отправляет события                         │
│                                                                     │
│  ───────────────────────────────────────────────────────────────    │
│                                                                     │
│  WEBSOCKET                                                          │
│  ════════════                                                       │
│  Client ◄═══════════════════════════════════════► Server            │
│         ◄─►  ◄─►  ◄─►  ◄─►  ◄─►  ◄─►  ◄─►                          │
│                                                                     │
│  Полнодуплексное соединение — оба могут отправлять в любой момент  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Детальное сравнение с Long Polling

```javascript
// SHORT POLLING
setInterval(async () => {
    const response = await fetch('/api/updates');
    // Сервер СРАЗУ отвечает
    processData(await response.json());
}, 5000);

// LONG POLLING
async function longPoll() {
    const response = await fetch('/api/updates?timeout=30');
    // Сервер ЖДЁТ до 30 секунд, пока не появятся данные
    processData(await response.json());
    longPoll(); // Сразу следующий запрос
}
longPoll();
```

**Ключевое отличие**: В Long Polling сервер держит соединение открытым, пока не появятся данные или не истечёт таймаут.

### Детальное сравнение с SSE

```javascript
// SHORT POLLING
setInterval(() => fetch('/api/updates'), 5000);

// SSE
const eventSource = new EventSource('/api/stream');
eventSource.onmessage = (event) => {
    // Сервер сам отправляет события, когда они появляются
    processData(JSON.parse(event.data));
};
```

**Ключевое отличие**: SSE использует одно постоянное соединение; сервер push'ит данные без запросов клиента.

### Детальное сравнение с WebSocket

```javascript
// SHORT POLLING
// Только клиент → сервер (запрос), сервер → клиент (ответ)
setInterval(() => fetch('/api/updates'), 5000);

// WEBSOCKET
const ws = new WebSocket('wss://example.com/ws');

// Двунаправленная связь
ws.onmessage = (event) => processData(event.data);
ws.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));
```

**Ключевое отличие**: WebSocket — полнодуплексный протокол; обе стороны могут отправлять данные в любой момент.

### Дерево принятия решений

```
                    Нужен real-time?
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
             НЕТ                   ДА
              │                     │
              │              Нужна двунаправленная
              │                    связь?
              │                     │
              │          ┌──────────┴──────────┐
              │          ▼                     ▼
              │         НЕТ                   ДА
              │          │                     │
              │    Много клиентов?      ───► WebSocket
              │          │
              │    ┌─────┴─────┐
              │    ▼           ▼
              │   НЕТ         ДА
              │    │           │
              │    │     Поддержка SSE?
              │    │           │
              │    │     ┌─────┴─────┐
              │    │     ▼           ▼
              │    │    ДА          НЕТ
              │    │     │           │
              │    │ ───►SSE    ───►Long Polling
              │    │
              │    └───► Long Polling или SSE
              │
              └───────────► Short Polling
                            (самый простой вариант)
```

---

## Оптимизация Short Polling

Если вы всё же решили использовать Short Polling, вот способы его оптимизации:

### 1. Адаптивный интервал

```javascript
class AdaptivePolling {
    constructor(url) {
        this.url = url;
        this.baseInterval = 2000;    // 2 секунды
        this.maxInterval = 30000;    // 30 секунд
        this.currentInterval = this.baseInterval;
        this.emptyResponses = 0;
    }

    async poll() {
        const data = await fetch(this.url).then(r => r.json());

        if (data.updates.length > 0) {
            // Есть данные — быстрый polling
            this.currentInterval = this.baseInterval;
            this.emptyResponses = 0;
        } else {
            // Нет данных — увеличиваем интервал
            this.emptyResponses++;
            if (this.emptyResponses > 5) {
                this.currentInterval = Math.min(
                    this.currentInterval * 1.5,
                    this.maxInterval
                );
            }
        }

        setTimeout(() => this.poll(), this.currentInterval);
    }
}
```

### 2. HTTP-кеширование

```python
# Flask сервер с кешированием
from flask import Flask, jsonify, request, make_response
from hashlib import md5

@app.route('/api/updates')
def get_updates():
    data = get_current_updates()

    # Генерируем ETag на основе данных
    etag = md5(str(data).encode()).hexdigest()

    # Проверяем If-None-Match от клиента
    if request.headers.get('If-None-Match') == etag:
        return '', 304  # Not Modified — экономим трафик

    response = make_response(jsonify(data))
    response.headers['ETag'] = etag
    response.headers['Cache-Control'] = 'private, max-age=0, must-revalidate'

    return response
```

### 3. Сжатие данных

```python
# FastAPI с gzip-сжатием
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()
app.add_middleware(GZipMiddleware, minimum_size=100)
```

---

## Заключение

**Short Polling** — это базовый метод получения обновлений от сервера, который:

- **Прост в реализации** — обычные HTTP-запросы через `setInterval`
- **Универсален** — работает везде без специальной инфраструктуры
- **Подходит для простых случаев** — админки, мониторинг с редкими обновлениями

Однако для современных real-time приложений лучше использовать:
- **Long Polling** — если нужна лучшая отзывчивость
- **SSE** — для односторонней потоковой передачи данных
- **WebSocket** — для полноценного двустороннего общения

Выбор метода зависит от конкретных требований: количества пользователей, важности задержки, ограничений инфраструктуры и сложности реализации.

---

## Дополнительные ресурсы

- [MDN: Fetch API](https://developer.mozilla.org/ru/docs/Web/API/Fetch_API)
- [MDN: setInterval](https://developer.mozilla.org/ru/docs/Web/API/setInterval)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Flask Documentation](https://flask.palletsprojects.com/)
