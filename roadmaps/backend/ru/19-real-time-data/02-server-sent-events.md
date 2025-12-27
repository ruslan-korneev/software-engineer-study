# Server-Sent Events (События, отправляемые сервером)

## Введение

**Server-Sent Events (SSE)** — это технология, позволяющая серверу отправлять данные клиенту через HTTP-соединение в режиме реального времени. SSE является частью спецификации HTML5 и предоставляет простой, стандартизированный способ push-уведомлений от сервера к браузеру.

### Ключевые характеристики SSE

- **Однонаправленная связь**: данные передаются только от сервера к клиенту
- **Использует HTTP**: работает поверх обычного HTTP/HTTPS соединения
- **Автоматическое переподключение**: браузер автоматически восстанавливает соединение при разрыве
- **Простота реализации**: не требует специальных протоколов или библиотек
- **Текстовый формат**: данные передаются в виде текста (обычно JSON)

### Отличие SSE от WebSockets

| Характеристика | SSE | WebSockets |
|----------------|-----|------------|
| Направление связи | Только сервер → клиент | Двунаправленная (сервер ↔ клиент) |
| Протокол | HTTP/HTTPS | WS/WSS (собственный протокол) |
| Формат данных | Только текст | Текст и бинарные данные |
| Автопереподключение | Встроенное | Требует реализации |
| Поддержка браузерами | Все современные (кроме IE) | Все современные |
| Сложность реализации | Простая | Средняя |
| Работа через прокси | Обычно без проблем | Может требовать настройки |
| Нагрузка на сервер | Низкая | Средняя |

SSE идеально подходит для сценариев, где клиенту нужно только получать обновления, а не отправлять данные в реальном времени.

---

## Как работает SSE

### Механизм работы

SSE использует длительное HTTP-соединение (long-lived connection), через которое сервер может отправлять события клиенту. Клиент инициирует соединение, а сервер держит его открытым и периодически отправляет данные.

```
┌─────────────┐                          ┌─────────────┐
│             │   GET /events            │             │
│             │ ─────────────────────►   │             │
│             │   Accept: text/event-    │             │
│             │   stream                 │             │
│             │                          │             │
│   Клиент    │   HTTP 200 OK            │   Сервер    │
│  (Браузер)  │   Content-Type:          │             │
│             │   text/event-stream      │             │
│             │ ◄─────────────────────   │             │
│             │                          │             │
│             │   data: {"msg": "Hi"}    │             │
│             │ ◄─────────────────────   │             │
│             │                          │             │
│             │   data: {"count": 1}     │             │
│             │ ◄─────────────────────   │             │
│             │                          │             │
│             │   data: {"count": 2}     │             │
│             │ ◄─────────────────────   │             │
│             │          ...             │             │
│             │   (соединение открыто)   │             │
└─────────────┘                          └─────────────┘
```

### EventSource API

Браузеры предоставляют встроенный API `EventSource` для работы с SSE:

```javascript
// Создание подключения к SSE-endpoint
const eventSource = new EventSource('/api/events');

// Обработка входящих сообщений (событие 'message')
eventSource.onmessage = function(event) {
    console.log('Получено сообщение:', event.data);
};

// Обработка открытия соединения
eventSource.onopen = function(event) {
    console.log('Соединение установлено');
};

// Обработка ошибок
eventSource.onerror = function(event) {
    console.log('Ошибка соединения:', event);
    if (eventSource.readyState === EventSource.CLOSED) {
        console.log('Соединение закрыто');
    }
};

// Закрытие соединения
eventSource.close();
```

### Состояния EventSource

`EventSource` имеет три состояния (`readyState`):

| Состояние | Значение | Описание |
|-----------|----------|----------|
| `CONNECTING` | 0 | Соединение устанавливается или переподключается |
| `OPEN` | 1 | Соединение активно, данные могут поступать |
| `CLOSED` | 2 | Соединение закрыто и не будет переподключаться |

### Формат сообщений SSE

Сервер отправляет данные в специальном текстовом формате. Каждое сообщение состоит из одного или нескольких полей, разделённых символом новой строки:

```
field: value\n
field: value\n
\n
```

Два символа новой строки (`\n\n`) означают конец сообщения.

#### Доступные поля

| Поле | Описание | Пример |
|------|----------|--------|
| `data` | Данные сообщения | `data: Hello World` |
| `event` | Тип события (по умолчанию 'message') | `event: notification` |
| `id` | Идентификатор события (для восстановления) | `id: 12345` |
| `retry` | Интервал переподключения в миллисекундах | `retry: 5000` |

#### Примеры сообщений

**Простое сообщение:**
```
data: Привет, мир!

```

**JSON-данные:**
```
data: {"user": "Иван", "action": "login"}

```

**Многострочные данные:**
```
data: Строка 1
data: Строка 2
data: Строка 3

```

**Именованное событие:**
```
event: userJoined
data: {"userId": 123, "name": "Мария"}

```

**Сообщение с идентификатором:**
```
id: 42
event: update
data: {"temperature": 23.5}

```

**Установка интервала переподключения:**
```
retry: 10000
data: Переподключение через 10 секунд при разрыве

```

### Восстановление соединения

Когда соединение разрывается, браузер автоматически пытается переподключиться. При этом он отправляет заголовок `Last-Event-ID` с последним полученным идентификатором события:

```
┌─────────────┐                          ┌─────────────┐
│             │   Первое подключение     │             │
│             │ ─────────────────────►   │             │
│             │                          │             │
│             │   id: 1                  │             │
│             │   data: Event 1          │             │
│             │ ◄─────────────────────   │             │
│             │                          │             │
│   Клиент    │   id: 2                  │   Сервер    │
│             │   data: Event 2          │             │
│             │ ◄─────────────────────   │             │
│             │                          │             │
│             │   ✖ Соединение разорвано │             │
│             │                          │             │
│             │   Переподключение        │             │
│             │   Last-Event-ID: 2       │             │
│             │ ─────────────────────►   │             │
│             │                          │             │
│             │   id: 3                  │             │
│             │   data: Event 3          │             │
│             │ ◄─────────────────────   │             │
└─────────────┘                          └─────────────┘
```

---

## Примеры кода

### JavaScript клиент

#### Базовый пример

```javascript
class SSEClient {
    constructor(url, options = {}) {
        this.url = url;
        this.options = options;
        this.eventSource = null;
        this.handlers = new Map();
    }

    // Подключение к SSE-endpoint
    connect() {
        this.eventSource = new EventSource(this.url, this.options);

        // Обработка стандартного события 'message'
        this.eventSource.onmessage = (event) => {
            this.handleEvent('message', event);
        };

        // Обработка открытия соединения
        this.eventSource.onopen = (event) => {
            console.log('[SSE] Соединение установлено');
            this.handleEvent('open', event);
        };

        // Обработка ошибок
        this.eventSource.onerror = (event) => {
            console.error('[SSE] Ошибка:', event);
            this.handleEvent('error', event);
        };

        return this;
    }

    // Подписка на определённый тип события
    on(eventType, callback) {
        // Добавляем обработчик в Map
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);
        }
        this.handlers.get(eventType).push(callback);

        // Для пользовательских событий добавляем слушатель
        if (!['message', 'open', 'error'].includes(eventType)) {
            this.eventSource.addEventListener(eventType, (event) => {
                this.handleEvent(eventType, event);
            });
        }

        return this;
    }

    // Вызов обработчиков события
    handleEvent(eventType, event) {
        const handlers = this.handlers.get(eventType) || [];
        handlers.forEach(handler => {
            try {
                // Пытаемся распарсить JSON
                const data = event.data ? JSON.parse(event.data) : event;
                handler(data, event);
            } catch (e) {
                // Если не JSON, передаём как строку
                handler(event.data, event);
            }
        });
    }

    // Закрытие соединения
    disconnect() {
        if (this.eventSource) {
            this.eventSource.close();
            console.log('[SSE] Соединение закрыто');
        }
    }

    // Получение состояния соединения
    get state() {
        if (!this.eventSource) return 'DISCONNECTED';
        const states = ['CONNECTING', 'OPEN', 'CLOSED'];
        return states[this.eventSource.readyState];
    }
}

// Использование
const client = new SSEClient('/api/events');

client
    .connect()
    .on('open', () => {
        console.log('Готов к получению событий');
    })
    .on('message', (data) => {
        console.log('Сообщение:', data);
    })
    .on('notification', (data) => {
        showNotification(data.title, data.body);
    })
    .on('error', () => {
        console.log('Переподключение...');
    });

// Закрытие при уходе со страницы
window.addEventListener('beforeunload', () => {
    client.disconnect();
});
```

#### React Hook для SSE

```javascript
import { useState, useEffect, useCallback, useRef } from 'react';

function useSSE(url, options = {}) {
    const [data, setData] = useState(null);
    const [error, setError] = useState(null);
    const [isConnected, setIsConnected] = useState(false);
    const eventSourceRef = useRef(null);

    const connect = useCallback(() => {
        // Закрываем предыдущее соединение
        if (eventSourceRef.current) {
            eventSourceRef.current.close();
        }

        const eventSource = new EventSource(url);
        eventSourceRef.current = eventSource;

        eventSource.onopen = () => {
            setIsConnected(true);
            setError(null);
        };

        eventSource.onmessage = (event) => {
            try {
                const parsed = JSON.parse(event.data);
                setData(parsed);
            } catch {
                setData(event.data);
            }
        };

        eventSource.onerror = (err) => {
            setError(err);
            setIsConnected(false);
        };

        return eventSource;
    }, [url]);

    const disconnect = useCallback(() => {
        if (eventSourceRef.current) {
            eventSourceRef.current.close();
            eventSourceRef.current = null;
            setIsConnected(false);
        }
    }, []);

    useEffect(() => {
        const eventSource = connect();

        return () => {
            eventSource.close();
        };
    }, [connect]);

    return { data, error, isConnected, connect, disconnect };
}

// Использование в компоненте
function NotificationsFeed() {
    const { data, isConnected, error } = useSSE('/api/notifications');

    if (error) {
        return <div>Ошибка подключения</div>;
    }

    return (
        <div>
            <div>Статус: {isConnected ? '🟢 Подключено' : '🔴 Отключено'}</div>
            {data && (
                <div className="notification">
                    <h3>{data.title}</h3>
                    <p>{data.message}</p>
                </div>
            )}
        </div>
    );
}
```

### Python сервер (FastAPI)

#### Базовый SSE endpoint

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from typing import AsyncGenerator
import asyncio
import json
from datetime import datetime

app = FastAPI()

async def event_generator() -> AsyncGenerator[str, None]:
    """Генератор событий SSE"""
    counter = 0
    while True:
        counter += 1

        # Формируем сообщение в формате SSE
        data = {
            "counter": counter,
            "timestamp": datetime.now().isoformat(),
            "message": f"Событие #{counter}"
        }

        # Формат SSE: "data: <данные>\n\n"
        yield f"data: {json.dumps(data, ensure_ascii=False)}\n\n"

        # Задержка между сообщениями
        await asyncio.sleep(1)

@app.get("/events")
async def sse_endpoint():
    """SSE endpoint для потоковой передачи событий"""
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Отключаем буферизацию nginx
        }
    )
```

#### Продвинутый SSE сервер с именованными событиями

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from typing import AsyncGenerator, Optional, Dict, List
from dataclasses import dataclass, field
from datetime import datetime
import asyncio
import json
import uuid

app = FastAPI()

# Настройка CORS для SSE
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["GET"],
    allow_headers=["*"],
)

@dataclass
class SSEMessage:
    """Структура SSE-сообщения"""
    data: str
    event: Optional[str] = None
    id: Optional[str] = None
    retry: Optional[int] = None

    def format(self) -> str:
        """Форматирование сообщения в SSE-формат"""
        lines = []

        if self.id:
            lines.append(f"id: {self.id}")
        if self.event:
            lines.append(f"event: {self.event}")
        if self.retry:
            lines.append(f"retry: {self.retry}")

        # Данные могут быть многострочными
        for line in self.data.split('\n'):
            lines.append(f"data: {line}")

        # Завершаем сообщение пустой строкой
        lines.append("")
        lines.append("")

        return "\n".join(lines)

class ConnectionManager:
    """Менеджер SSE-подключений"""

    def __init__(self):
        self.connections: Dict[str, asyncio.Queue] = {}

    async def connect(self, client_id: str) -> asyncio.Queue:
        """Регистрация нового подключения"""
        queue = asyncio.Queue()
        self.connections[client_id] = queue
        print(f"[SSE] Клиент подключён: {client_id}")
        return queue

    def disconnect(self, client_id: str):
        """Удаление подключения"""
        if client_id in self.connections:
            del self.connections[client_id]
            print(f"[SSE] Клиент отключён: {client_id}")

    async def send_to_client(self, client_id: str, message: SSEMessage):
        """Отправка сообщения конкретному клиенту"""
        if client_id in self.connections:
            await self.connections[client_id].put(message)

    async def broadcast(self, message: SSEMessage):
        """Отправка сообщения всем подключённым клиентам"""
        for queue in self.connections.values():
            await queue.put(message)

    @property
    def active_connections(self) -> int:
        return len(self.connections)

manager = ConnectionManager()

async def event_stream(
    client_id: str,
    queue: asyncio.Queue,
    last_event_id: Optional[str] = None
) -> AsyncGenerator[str, None]:
    """Генератор потока событий для клиента"""

    # Отправляем приветственное сообщение
    welcome = SSEMessage(
        data=json.dumps({
            "type": "connected",
            "client_id": client_id,
            "timestamp": datetime.now().isoformat()
        }),
        event="system",
        id=str(uuid.uuid4()),
        retry=5000  # Переподключаться через 5 секунд
    )
    yield welcome.format()

    # Если есть last_event_id, можно восстановить пропущенные события
    if last_event_id:
        # Здесь можно реализовать логику восстановления
        recovery = SSEMessage(
            data=json.dumps({"recovered_from": last_event_id}),
            event="recovery"
        )
        yield recovery.format()

    try:
        while True:
            # Ждём сообщение из очереди с таймаутом
            try:
                message = await asyncio.wait_for(
                    queue.get(),
                    timeout=30.0  # Heartbeat каждые 30 секунд
                )
                yield message.format()
            except asyncio.TimeoutError:
                # Отправляем heartbeat для поддержания соединения
                heartbeat = SSEMessage(
                    data="",
                    event="heartbeat"
                )
                yield f": heartbeat\n\n"  # Комментарий SSE
    except asyncio.CancelledError:
        pass
    finally:
        manager.disconnect(client_id)

@app.get("/events")
async def sse_events(request: Request):
    """Основной SSE endpoint"""

    # Генерируем или получаем ID клиента
    client_id = str(uuid.uuid4())

    # Получаем Last-Event-ID для восстановления
    last_event_id = request.headers.get("Last-Event-ID")

    # Регистрируем подключение
    queue = await manager.connect(client_id)

    async def generate():
        async for event in event_stream(client_id, queue, last_event_id):
            # Проверяем, не отключился ли клиент
            if await request.is_disconnected():
                break
            yield event

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        }
    )

@app.post("/broadcast")
async def broadcast_message(message: dict):
    """Endpoint для отправки сообщения всем клиентам"""
    sse_message = SSEMessage(
        data=json.dumps(message, ensure_ascii=False),
        event=message.get("event", "message"),
        id=str(uuid.uuid4())
    )
    await manager.broadcast(sse_message)
    return {
        "status": "ok",
        "recipients": manager.active_connections
    }

@app.get("/stats")
async def get_stats():
    """Статистика подключений"""
    return {
        "active_connections": manager.active_connections
    }
```

### Python сервер (Flask)

```python
from flask import Flask, Response, request
from typing import Generator
import json
import time
from datetime import datetime
from queue import Queue, Empty
from threading import Lock
import uuid

app = Flask(__name__)

class SSEManager:
    """Менеджер подключений для Flask"""

    def __init__(self):
        self.clients: dict[str, Queue] = {}
        self.lock = Lock()

    def register(self, client_id: str) -> Queue:
        """Регистрация клиента"""
        with self.lock:
            queue = Queue()
            self.clients[client_id] = queue
            return queue

    def unregister(self, client_id: str):
        """Удаление клиента"""
        with self.lock:
            if client_id in self.clients:
                del self.clients[client_id]

    def broadcast(self, data: dict, event: str = None):
        """Рассылка всем клиентам"""
        message = self._format_sse(data, event)
        with self.lock:
            for queue in self.clients.values():
                queue.put(message)

    def _format_sse(self, data: dict, event: str = None) -> str:
        """Форматирование SSE-сообщения"""
        lines = []
        if event:
            lines.append(f"event: {event}")
        lines.append(f"id: {uuid.uuid4()}")
        lines.append(f"data: {json.dumps(data, ensure_ascii=False)}")
        lines.append("")
        lines.append("")
        return "\n".join(lines)

sse_manager = SSEManager()

def event_stream(client_id: str, queue: Queue) -> Generator[str, None, None]:
    """Генератор событий для Flask"""

    # Приветственное сообщение
    welcome = {
        "type": "connected",
        "client_id": client_id,
        "timestamp": datetime.now().isoformat()
    }
    yield f"event: system\ndata: {json.dumps(welcome)}\n\n"

    try:
        while True:
            try:
                # Ждём сообщение с таймаутом
                message = queue.get(timeout=30)
                yield message
            except Empty:
                # Heartbeat
                yield ": heartbeat\n\n"
    except GeneratorExit:
        sse_manager.unregister(client_id)

@app.route('/events')
def sse_endpoint():
    """SSE endpoint для Flask"""
    client_id = str(uuid.uuid4())
    queue = sse_manager.register(client_id)

    response = Response(
        event_stream(client_id, queue),
        mimetype='text/event-stream'
    )
    response.headers['Cache-Control'] = 'no-cache'
    response.headers['Connection'] = 'keep-alive'
    response.headers['X-Accel-Buffering'] = 'no'

    return response

@app.route('/send', methods=['POST'])
def send_event():
    """Отправка события всем клиентам"""
    data = request.json
    event_type = data.pop('_event', 'message')
    sse_manager.broadcast(data, event_type)
    return {'status': 'ok'}

if __name__ == '__main__':
    app.run(debug=True, threaded=True)
```

---

## Преимущества и недостатки

### Таблица сравнения

| Аспект | Преимущества | Недостатки |
|--------|--------------|------------|
| **Простота** | Простой API (EventSource), не требует библиотек | Только однонаправленная связь |
| **Протокол** | Использует стандартный HTTP/HTTPS | Не поддерживает бинарные данные |
| **Переподключение** | Автоматическое встроенное переподключение | Может создавать нагрузку при частых разрывах |
| **Совместимость** | Работает через прокси и файрволы | Не поддерживается в Internet Explorer |
| **Масштабирование** | Легко масштабируется горизонтально | Ограничение на количество соединений (6 на домен в HTTP/1.1) |
| **Отладка** | Легко отлаживать в DevTools | Нет стандартного способа отправки данных на сервер |
| **Ресурсы** | Меньше накладных расходов, чем WebSocket | Держит соединение открытым |
| **Восстановление** | Поддержка Last-Event-ID | Требует реализации на сервере |

### Преимущества в деталях

1. **Простота реализации**
   - Не нужны специальные библиотеки на клиенте
   - EventSource API интуитивно понятен
   - Сервер может быть реализован на любом языке

2. **Надёжность соединения**
   - Браузер автоматически переподключается
   - Поддержка восстановления через Last-Event-ID
   - Настраиваемый интервал переподключения

3. **Совместимость с инфраструктурой**
   - Работает через стандартные HTTP-прокси
   - Не требует специальной настройки файрволов
   - Поддерживается CDN и load balancer'ами

4. **Эффективность**
   - Меньше overhead по сравнению с polling
   - Одно постоянное соединение вместо множества запросов
   - Низкая латентность доставки сообщений

### Недостатки в деталях

1. **Ограничения протокола**
   - Только текстовые данные (нужна сериализация для бинарных)
   - Однонаправленная связь (для отправки нужен отдельный запрос)
   - Максимум 6 соединений на домен в HTTP/1.1

2. **Поддержка браузеров**
   - Не работает в Internet Explorer
   - Требуется polyfill для старых браузеров

3. **Серверные ресурсы**
   - Каждый клиент держит открытое соединение
   - Нужно следить за количеством подключений
   - Требуется правильная настройка таймаутов

---

## Когда использовать SSE

### Идеальные сценарии применения

#### 1. Системы уведомлений

```javascript
// Клиент подписывается на уведомления
const notifications = new EventSource('/api/notifications');

notifications.addEventListener('alert', (event) => {
    const data = JSON.parse(event.data);
    showNotification(data.title, data.message);
});

notifications.addEventListener('badge', (event) => {
    const data = JSON.parse(event.data);
    updateBadgeCount(data.count);
});
```

#### 2. Live-ленты и обновления контента

```javascript
// Живая лента новостей
const feed = new EventSource('/api/feed/live');

feed.onmessage = (event) => {
    const post = JSON.parse(event.data);
    prependToFeed(post);
};
```

#### 3. Мониторинг и дашборды

```python
# Сервер: отправка метрик в реальном времени
async def metrics_stream():
    while True:
        metrics = await collect_system_metrics()
        yield f"event: metrics\ndata: {json.dumps(metrics)}\n\n"
        await asyncio.sleep(5)
```

```javascript
// Клиент: отображение метрик
const monitoring = new EventSource('/api/metrics');

monitoring.addEventListener('metrics', (event) => {
    const metrics = JSON.parse(event.data);
    updateDashboard(metrics);
});
```

#### 4. Прогресс длительных операций

```python
# Сервер: отправка прогресса обработки
async def process_file(file_id: str):
    queue = get_client_queue(file_id)

    for i, chunk in enumerate(process_chunks(file_id)):
        progress = (i + 1) / total_chunks * 100
        await queue.put(SSEMessage(
            data=json.dumps({"progress": progress}),
            event="progress"
        ))

    await queue.put(SSEMessage(
        data=json.dumps({"status": "completed"}),
        event="complete"
    ))
```

#### 5. Стоимость акций / криптовалют

```javascript
const prices = new EventSource('/api/stocks/stream');

prices.addEventListener('price_update', (event) => {
    const { symbol, price, change } = JSON.parse(event.data);
    updateStockTicker(symbol, price, change);
});
```

#### 6. Чат (только для получения сообщений)

```javascript
// SSE для получения сообщений
const chat = new EventSource('/api/chat/room/123');

chat.addEventListener('message', (event) => {
    const msg = JSON.parse(event.data);
    appendMessage(msg);
});

// Отправка через обычный POST
async function sendMessage(text) {
    await fetch('/api/chat/room/123/messages', {
        method: 'POST',
        body: JSON.stringify({ text }),
        headers: { 'Content-Type': 'application/json' }
    });
}
```

### Когда НЕ использовать SSE

| Сценарий | Причина | Альтернатива |
|----------|---------|--------------|
| Двусторонняя связь в реальном времени | SSE только от сервера к клиенту | WebSockets |
| Передача бинарных данных | SSE только текстовые данные | WebSockets |
| Игры в реальном времени | Нужна низкая латентность в обе стороны | WebSockets / WebRTC |
| Видеозвонки | Требуется P2P и бинарные данные | WebRTC |
| Редкие обновления | Overhead постоянного соединения | Long polling / обычные запросы |

---

## Сравнение с WebSockets и Polling

### Архитектурное сравнение

```
┌─────────────────────────────────────────────────────────────────────┐
│                         POLLING                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Клиент          Сервер                                            │
│      │               │                                               │
│      │─── Запрос ───►│                                               │
│      │◄── Ответ ────│                                               │
│      │               │     (пауза)                                   │
│      │─── Запрос ───►│                                               │
│      │◄── Ответ ────│                                               │
│      │               │     (пауза)                                   │
│      │─── Запрос ───►│                                               │
│      │◄── Ответ ────│                                               │
│                                                                      │
│   Множество отдельных HTTP-запросов                                 │
│   Высокий overhead, задержка = интервал polling                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      LONG POLLING                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Клиент          Сервер                                            │
│      │               │                                               │
│      │─── Запрос ───►│                                               │
│      │               │     (ожидание данных...)                      │
│      │◄── Ответ ────│     (данные готовы)                           │
│      │─── Запрос ───►│                                               │
│      │               │     (ожидание данных...)                      │
│      │◄── Ответ ────│                                               │
│                                                                      │
│   Сервер держит запрос до появления данных                          │
│   Средний overhead, хорошая латентность                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    SERVER-SENT EVENTS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Клиент          Сервер                                            │
│      │               │                                               │
│      │─── GET ──────►│                                               │
│      │◄── Соединение установлено ──                                 │
│      │               │                                               │
│      │◄── Event 1 ──│                                               │
│      │◄── Event 2 ──│                                               │
│      │◄── Event 3 ──│                                               │
│      │       ...     │                                               │
│                                                                      │
│   Одно постоянное HTTP-соединение                                   │
│   Низкий overhead, мгновенная доставка                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       WEBSOCKETS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Клиент          Сервер                                            │
│      │               │                                               │
│      │─── Upgrade ──►│                                               │
│      │◄── Accept ───│                                               │
│      │               │                                               │
│      │◄─── Data ────│                                               │
│      │──── Data ───►│                                               │
│      │◄─── Data ────│                                               │
│      │──── Data ───►│                                               │
│                                                                      │
│   Полнодуплексное соединение                                        │
│   Минимальный overhead, двунаправленность                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Детальное сравнение

| Критерий | Polling | Long Polling | SSE | WebSockets |
|----------|---------|--------------|-----|------------|
| **Направление** | Клиент → Сервер | Клиент → Сервер | Сервер → Клиент | Двунаправленное |
| **Протокол** | HTTP | HTTP | HTTP | WS (поверх TCP) |
| **Латентность** | Высокая (интервал) | Средняя | Низкая | Очень низкая |
| **Overhead** | Высокий | Средний | Низкий | Минимальный |
| **Сложность сервера** | Простая | Средняя | Средняя | Высокая |
| **Сложность клиента** | Простая | Средняя | Простая (API) | Средняя |
| **Масштабирование** | Легко | Средне | Легко | Сложно |
| **Бинарные данные** | Да | Да | Нет | Да |
| **Работа через прокси** | Да | Да | Да | Может быть проблема |
| **Автопереподключение** | Нет | Нет | Да | Нет |
| **Поддержка IE** | Да | Да | Нет | Да (10+) |

### Когда что выбирать

```
                    ┌─────────────────────────────────────┐
                    │     Нужна двусторонняя связь?       │
                    └─────────────────┬───────────────────┘
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                        Да                        Нет
                         │                         │
                         ▼                         ▼
              ┌──────────────────┐     ┌──────────────────────┐
              │ Нужны бинарные   │     │ Частота обновлений?  │
              │ данные / игры?   │     └──────────┬───────────┘
              └────────┬─────────┘                │
                       │              ┌───────────┼───────────┐
              ┌────────┴────────┐     ▼           ▼           ▼
              ▼                 ▼   Редко      Часто     Постоянно
             Да               Нет    │           │           │
              │                 │    ▼           ▼           ▼
              ▼                 ▼  Polling  Long Polling    SSE
         WebSockets        WebSockets
```

### Пример выбора технологии

| Приложение | Рекомендация | Причина |
|------------|--------------|---------|
| Чат | WebSockets | Двунаправленная связь, низкая латентность |
| Уведомления | SSE | Только от сервера, автопереподключение |
| Онлайн-игра | WebSockets | Двунаправленность, бинарные данные |
| Биржевые котировки | SSE | Только получение данных |
| Дашборд мониторинга | SSE | Периодические обновления от сервера |
| Совместное редактирование | WebSockets | Синхронизация в обе стороны |
| Проверка статуса заказа | Long Polling / SSE | Редкие обновления |
| Email-клиент (новые письма) | SSE | Уведомления от сервера |

---

## Лучшие практики

### На сервере

1. **Устанавливайте правильные заголовки**
   ```python
   headers = {
       "Content-Type": "text/event-stream",
       "Cache-Control": "no-cache",
       "Connection": "keep-alive",
       "X-Accel-Buffering": "no",  # для nginx
   }
   ```

2. **Отправляйте heartbeat-сообщения**
   ```python
   # Каждые 15-30 секунд для поддержания соединения
   yield ": heartbeat\n\n"
   ```

3. **Используйте идентификаторы событий**
   ```python
   yield f"id: {uuid.uuid4()}\ndata: {data}\n\n"
   ```

4. **Обрабатывайте Last-Event-ID**
   ```python
   last_id = request.headers.get("Last-Event-ID")
   if last_id:
       # Восстановить пропущенные события
       pass
   ```

5. **Ограничивайте количество соединений**
   ```python
   MAX_CONNECTIONS = 1000
   if manager.active_connections >= MAX_CONNECTIONS:
       raise HTTPException(503, "Too many connections")
   ```

### На клиенте

1. **Обрабатывайте все состояния**
   ```javascript
   eventSource.onopen = () => { /* соединение открыто */ };
   eventSource.onerror = () => { /* обработка ошибки */ };
   eventSource.onmessage = () => { /* обработка данных */ };
   ```

2. **Закрывайте соединение при уходе**
   ```javascript
   window.addEventListener('beforeunload', () => {
       eventSource.close();
   });
   ```

3. **Используйте именованные события**
   ```javascript
   eventSource.addEventListener('notification', handler);
   eventSource.addEventListener('update', handler);
   ```

---

## Заключение

Server-Sent Events — это простая и эффективная технология для push-уведомлений от сервера к клиенту. Она идеально подходит для:

- Уведомлений и алертов
- Live-лент и обновлений контента
- Мониторинга и дашбордов
- Отслеживания прогресса операций

Главные преимущества SSE — простота реализации, автоматическое переподключение и работа через стандартный HTTP. Если вашему приложению нужна только однонаправленная связь (сервер → клиент), SSE часто является лучшим выбором по сравнению с WebSockets благодаря меньшей сложности и лучшей совместимости с инфраструктурой.
