# Server-Sent Events (SSE)

## Введение

**Server-Sent Events (SSE)** — это технология, позволяющая серверу отправлять обновления клиенту через HTTP-соединение в режиме реального времени. В отличие от WebSocket, SSE работает только в одном направлении — от сервера к клиенту, но при этом предоставляет более простой API и встроенную поддержку автопереподключения.

SSE стандартизирован как часть HTML5 и использует интерфейс `EventSource` на стороне клиента. Данные передаются в текстовом формате (text/event-stream).

---

## Принцип работы

### HTTP-запрос и ответ

Клиент отправляет обычный HTTP GET-запрос:

```
GET /events HTTP/1.1
Host: example.com
Accept: text/event-stream
Cache-Control: no-cache
```

Сервер отвечает с заголовком `Content-Type: text/event-stream` и оставляет соединение открытым:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: Первое сообщение

data: Второе сообщение

event: custom-event
data: {"key": "value"}

```

### Формат сообщений

SSE использует простой текстовый формат:

```
field: value\n
```

Основные поля:

| Поле | Описание |
|------|----------|
| `data` | Содержимое сообщения (может быть многострочным) |
| `event` | Тип события (по умолчанию "message") |
| `id` | Уникальный идентификатор события |
| `retry` | Интервал переподключения в миллисекундах |

### Примеры форматов сообщений

```
# Простое сообщение
data: Hello, World!

# Многострочное сообщение
data: Первая строка
data: Вторая строка
data: Третья строка

# Сообщение с ID и типом события
id: 123
event: user-login
data: {"userId": 42, "username": "ivan"}

# Установка интервала переподключения
retry: 5000
data: Переподключение через 5 секунд

```

**Важно:** Каждое событие отделяется двумя символами новой строки (`\n\n`).

---

## Сравнение с альтернативами

### SSE vs WebSocket

| Характеристика | SSE | WebSocket |
|---------------|-----|-----------|
| Направление | Только сервер -> клиент | Двунаправленное |
| Протокол | HTTP | ws:// / wss:// |
| Формат данных | Только текст | Текст + бинарные |
| Автопереподключение | Встроено | Нужно реализовать |
| Поддержка HTTP/2 | Отличная (мультиплексирование) | Отдельное соединение |
| Прокси/файрволы | Работает через HTTP | Может блокироваться |
| Сложность | Низкая | Средняя |

### SSE vs Long Polling

| Характеристика | SSE | Long Polling |
|---------------|-----|--------------|
| Постоянное соединение | Да | Нет (переподключение) |
| Нативный API | EventSource | Ручная реализация |
| Эффективность | Высокая | Низкая (overhead) |
| ID событий | Встроено | Нужно реализовать |

### Когда использовать SSE?

**Используйте SSE когда:**
- Нужен только односторонний поток данных (сервер -> клиент)
- Важна простота реализации
- Требуется автоматическое переподключение
- Работаете через прокси/корпоративные файрволы
- Используете HTTP/2 (эффективное мультиплексирование)

**Используйте WebSocket когда:**
- Нужна двунаправленная связь
- Требуется передача бинарных данных
- Важна минимальная задержка
- Высокая частота сообщений в обоих направлениях

---

## Практические примеры кода

### Python (Flask)

```python
from flask import Flask, Response
import time
import json

app = Flask(__name__)

def generate_events():
    """Генератор событий SSE"""
    event_id = 0

    while True:
        event_id += 1

        # Формируем SSE сообщение
        data = {
            "id": event_id,
            "timestamp": time.time(),
            "message": f"Событие #{event_id}"
        }

        # Формат: id, event, data
        yield f"id: {event_id}\n"
        yield f"event: update\n"
        yield f"data: {json.dumps(data)}\n\n"

        time.sleep(2)  # Отправляем событие каждые 2 секунды

@app.route('/events')
def sse_events():
    return Response(
        generate_events(),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'X-Accel-Buffering': 'no'  # Для Nginx
        }
    )

@app.route('/')
def index():
    return '''
    <!DOCTYPE html>
    <html>
    <body>
        <h1>SSE Demo</h1>
        <div id="events"></div>
        <script>
            const evtSource = new EventSource('/events');
            evtSource.addEventListener('update', function(event) {
                const data = JSON.parse(event.data);
                document.getElementById('events').innerHTML +=
                    '<p>Event #' + data.id + ': ' + data.message + '</p>';
            });
        </script>
    </body>
    </html>
    '''

if __name__ == '__main__':
    app.run(threaded=True)
```

### Python (FastAPI)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse
import asyncio
import json
from datetime import datetime

app = FastAPI()

# Хранилище подписчиков
subscribers = set()

async def event_generator(request: Request):
    """Асинхронный генератор событий"""
    client_id = id(request)
    subscribers.add(client_id)

    try:
        counter = 0
        while True:
            # Проверяем, не отключился ли клиент
            if await request.is_disconnected():
                break

            counter += 1
            data = {
                "counter": counter,
                "timestamp": datetime.now().isoformat(),
                "subscribers_count": len(subscribers)
            }

            yield {
                "event": "message",
                "id": str(counter),
                "data": json.dumps(data)
            }

            await asyncio.sleep(1)
    finally:
        subscribers.discard(client_id)

@app.get("/stream")
async def stream(request: Request):
    return EventSourceResponse(event_generator(request))

# Публикация событий для всех подписчиков
event_queue = asyncio.Queue()

async def notification_generator(request: Request):
    """Генератор для уведомлений"""
    while True:
        if await request.is_disconnected():
            break

        try:
            # Ждём событие с таймаутом
            event = await asyncio.wait_for(
                event_queue.get(),
                timeout=30
            )
            yield {
                "event": "notification",
                "data": json.dumps(event)
            }
        except asyncio.TimeoutError:
            # Отправляем heartbeat
            yield {"event": "heartbeat", "data": ""}

@app.get("/notifications")
async def notifications(request: Request):
    return EventSourceResponse(notification_generator(request))

@app.post("/publish")
async def publish(message: dict):
    """Публикация события всем подписчикам"""
    await event_queue.put(message)
    return {"status": "published"}
```

### Python (aiohttp)

```python
from aiohttp import web
import asyncio
import json

async def sse_handler(request):
    """SSE endpoint с использованием aiohttp"""
    response = web.StreamResponse(
        status=200,
        headers={
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
        }
    )
    await response.prepare(request)

    counter = 0
    try:
        while True:
            counter += 1

            # Формируем SSE событие
            event = f"id: {counter}\n"
            event += f"event: tick\n"
            event += f"data: {json.dumps({'count': counter})}\n\n"

            await response.write(event.encode('utf-8'))
            await asyncio.sleep(1)

    except asyncio.CancelledError:
        pass

    return response

app = web.Application()
app.router.add_get('/events', sse_handler)

if __name__ == '__main__':
    web.run_app(app, port=8080)
```

### JavaScript (клиент в браузере)

```javascript
class SSEClient {
    constructor(url) {
        this.url = url;
        this.eventSource = null;
        this.handlers = new Map();
    }

    connect() {
        this.eventSource = new EventSource(this.url);

        // Обработка стандартного события message
        this.eventSource.onmessage = (event) => {
            console.log('Получено сообщение:', event.data);
            this.emit('message', JSON.parse(event.data));
        };

        // Обработка открытия соединения
        this.eventSource.onopen = () => {
            console.log('SSE соединение установлено');
            this.emit('connected');
        };

        // Обработка ошибок (включая автопереподключение)
        this.eventSource.onerror = (error) => {
            console.error('SSE ошибка:', error);
            if (this.eventSource.readyState === EventSource.CLOSED) {
                console.log('Соединение закрыто');
                this.emit('disconnected');
            } else {
                console.log('Попытка переподключения...');
                this.emit('reconnecting');
            }
        };
    }

    // Подписка на кастомные события
    on(eventType, callback) {
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);

            // Добавляем слушатель для SSE
            if (this.eventSource) {
                this.eventSource.addEventListener(eventType, (event) => {
                    const data = JSON.parse(event.data);
                    this.handlers.get(eventType).forEach(cb => cb(data, event));
                });
            }
        }
        this.handlers.get(eventType).push(callback);
    }

    emit(eventType, data) {
        if (this.handlers.has(eventType)) {
            this.handlers.get(eventType).forEach(cb => cb(data));
        }
    }

    close() {
        if (this.eventSource) {
            this.eventSource.close();
            console.log('SSE соединение закрыто');
        }
    }
}

// Использование
const sse = new SSEClient('/events');

sse.on('connected', () => {
    console.log('Подключено к серверу');
});

sse.on('message', (data) => {
    console.log('Новое сообщение:', data);
});

// Кастомные события
sse.on('notification', (data) => {
    showNotification(data.title, data.body);
});

sse.on('update', (data) => {
    updateDashboard(data);
});

sse.connect();
```

### JavaScript (Node.js сервер с Express)

```javascript
const express = require('express');
const app = express();

// Хранилище подключённых клиентов
const clients = new Set();

// SSE endpoint
app.get('/events', (req, res) => {
    // Устанавливаем заголовки для SSE
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('X-Accel-Buffering', 'no'); // Для Nginx

    // Отправляем начальное сообщение
    res.write(`data: ${JSON.stringify({ type: 'connected' })}\n\n`);

    // Добавляем клиента
    clients.add(res);
    console.log(`Клиент подключился. Всего: ${clients.size}`);

    // Heartbeat каждые 30 секунд
    const heartbeatInterval = setInterval(() => {
        res.write(': heartbeat\n\n');
    }, 30000);

    // Обработка отключения
    req.on('close', () => {
        clients.delete(res);
        clearInterval(heartbeatInterval);
        console.log(`Клиент отключился. Всего: ${clients.size}`);
    });
});

// Функция отправки события всем клиентам
function broadcast(event, data) {
    const message = `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`;
    clients.forEach(client => {
        client.write(message);
    });
}

// Endpoint для отправки уведомлений
app.post('/notify', express.json(), (req, res) => {
    const { event, data } = req.body;
    broadcast(event || 'notification', data);
    res.json({ sent: clients.size });
});

// Пример: отправка обновлений каждые 5 секунд
let counter = 0;
setInterval(() => {
    counter++;
    broadcast('update', {
        counter,
        timestamp: new Date().toISOString()
    });
}, 5000);

app.listen(3000, () => {
    console.log('SSE сервер запущен на http://localhost:3000');
});
```

---

## Примеры использования в реальных приложениях

### 1. Новостные ленты и социальные сети
- Обновления ленты в реальном времени
- Уведомления о новых постах, лайках, комментариях
- Счётчики непрочитанных сообщений

### 2. Биржевые котировки
- Стриминг цен акций
- Обновления курсов валют
- Графики в реальном времени

### 3. Мониторинг и дашборды
- Метрики серверов
- Логи в реальном времени
- CI/CD pipeline статусы

### 4. Прогресс длительных операций
```python
@app.get("/upload-progress/{task_id}")
async def upload_progress(task_id: str, request: Request):
    async def generate():
        while True:
            progress = get_task_progress(task_id)
            yield {
                "event": "progress",
                "data": json.dumps({"percent": progress})
            }

            if progress >= 100:
                yield {
                    "event": "complete",
                    "data": json.dumps({"status": "done"})
                }
                break

            await asyncio.sleep(0.5)

    return EventSourceResponse(generate())
```

### 5. Живые спортивные результаты
- Обновления счёта
- Статистика матча
- Комментарии в реальном времени

### 6. Системы оповещения
- Уведомления о безопасности
- Оповещения о важных событиях
- Системные сообщения

---

## Best Practices

### 1. Heartbeat для поддержания соединения
```python
async def event_generator():
    last_event_time = time.time()

    while True:
        # Проверяем наличие новых событий
        event = await get_next_event()

        if event:
            yield format_sse(event)
            last_event_time = time.time()
        elif time.time() - last_event_time > 15:
            # Отправляем heartbeat каждые 15 секунд
            yield ": heartbeat\n\n"
            last_event_time = time.time()

        await asyncio.sleep(0.1)
```

### 2. Использование Event ID для восстановления
```python
@app.get("/events")
async def events(request: Request):
    # Получаем Last-Event-ID из заголовка
    last_id = request.headers.get('Last-Event-ID', '0')

    async def generate():
        # Начинаем с пропущенных событий
        for event in get_events_after(last_id):
            yield format_sse(event)

        # Затем стримим новые
        async for event in stream_new_events():
            yield format_sse(event)

    return EventSourceResponse(generate())
```

### 3. Настройка retry интервала
```python
def format_sse(event):
    # Устанавливаем интервал переподключения 5 секунд
    message = f"retry: 5000\n"
    message += f"id: {event['id']}\n"
    message += f"event: {event['type']}\n"
    message += f"data: {json.dumps(event['data'])}\n\n"
    return message
```

### 4. Graceful shutdown
```python
import signal

clients = set()

async def cleanup():
    for client in clients:
        await client.write("event: shutdown\ndata: Server shutting down\n\n")
        await client.close()

def handle_shutdown(signum, frame):
    asyncio.create_task(cleanup())

signal.signal(signal.SIGTERM, handle_shutdown)
```

### 5. Группировка клиентов по каналам
```python
channels = defaultdict(set)

@app.get("/events/{channel}")
async def channel_events(channel: str, request: Request):
    async def generate():
        queue = asyncio.Queue()
        channels[channel].add(queue)

        try:
            while True:
                event = await queue.get()
                yield format_sse(event)
        finally:
            channels[channel].discard(queue)

    return EventSourceResponse(generate())

async def publish(channel: str, event: dict):
    for queue in channels[channel]:
        await queue.put(event)
```

---

## Типичные ошибки

### 1. Буферизация ответа
```python
# Плохо - данные буферизируются
@app.route('/events')
def events():
    return Response(generate(), mimetype='text/event-stream')

# Хорошо - отключаем буферизацию
@app.route('/events')
def events():
    return Response(
        generate(),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'X-Accel-Buffering': 'no'  # Для Nginx
        }
    )
```

### 2. Неправильный формат сообщений
```python
# Плохо - нет двойного перевода строки
yield f"data: {message}\n"

# Хорошо - каждое событие заканчивается \n\n
yield f"data: {message}\n\n"

# Плохо - многострочные данные одной строкой
yield f"data: line1\nline2\nline3\n\n"

# Хорошо - каждая строка с префиксом data:
yield f"data: line1\ndata: line2\ndata: line3\n\n"
```

### 3. Игнорирование Last-Event-ID
```javascript
// Плохо - не обрабатываем восстановление
const evtSource = new EventSource('/events');

// Хорошо - сервер должен проверять Last-Event-ID
// и отправлять пропущенные события
```

### 4. Утечка ресурсов
```python
# Плохо - клиенты не удаляются при отключении
@app.get("/events")
async def events():
    clients.add(client)
    async for event in stream():
        yield event

# Хорошо - используем try/finally
@app.get("/events")
async def events(request: Request):
    client = create_client()
    clients.add(client)

    try:
        async for event in stream():
            if await request.is_disconnected():
                break
            yield event
    finally:
        clients.remove(client)
```

### 5. Отсутствие проверки отключения клиента
```python
# Плохо - не проверяем, подключён ли клиент
async def generate():
    while True:
        yield format_sse(get_event())
        await asyncio.sleep(1)

# Хорошо - проверяем статус соединения
async def generate(request):
    while True:
        if await request.is_disconnected():
            break
        yield format_sse(get_event())
        await asyncio.sleep(1)
```

### 6. Проблемы с CORS
```python
# Добавляем CORS заголовки для SSE
@app.after_request
def add_cors_headers(response):
    response.headers['Access-Control-Allow-Origin'] = '*'
    response.headers['Access-Control-Allow-Headers'] = 'Content-Type'
    return response
```

---

## Ограничения SSE

### 1. Лимит соединений браузера
Браузеры ограничивают количество одновременных HTTP-соединений к одному домену (обычно 6). Решение — использовать HTTP/2, который мультиплексирует соединения.

### 2. Только текстовые данные
SSE поддерживает только текст. Для бинарных данных используйте base64 или WebSocket.

### 3. Однонаправленность
SSE работает только от сервера к клиенту. Для отправки данных на сервер используйте обычные HTTP-запросы.

### 4. Поддержка браузеров
SSE не поддерживается в Internet Explorer. Для поддержки используйте полифилл.

```javascript
// Полифилл для старых браузеров
if (!window.EventSource) {
    // Используем eventsource-polyfill
    // npm install eventsource-polyfill
}
```

---

## Заключение

Server-Sent Events — отличный выбор для приложений, требующих push-уведомлений от сервера к клиенту. Технология проста в реализации, имеет встроенную поддержку автопереподключения и отлично работает через HTTP-прокси. Используйте SSE для новостных лент, уведомлений, мониторинга и других сценариев, где достаточно одностороннего потока данных.

Для двунаправленной связи или передачи бинарных данных рассмотрите использование WebSocket.
