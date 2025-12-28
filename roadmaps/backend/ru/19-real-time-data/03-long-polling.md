# Long Polling (Длинный опрос)

[prev: 02-server-sent-events](./02-server-sent-events.md) | [next: 04-short-polling](./04-short-polling.md)

---

## Введение

**Long Polling** — это техника взаимодействия клиента с сервером, которая позволяет получать обновления данных в режиме, близком к реальному времени. В отличие от обычного (short) polling, где клиент постоянно отправляет запросы с фиксированным интервалом, long polling держит соединение открытым до тех пор, пока сервер не получит новые данные для отправки.

### Зачем нужен Long Polling?

Long Polling был разработан как решение проблемы получения данных в реальном времени до появления WebSocket. Основные причины использования:

1. **Уменьшение нагрузки на сервер** — вместо множества пустых запросов, клиент ждёт ответа с данными
2. **Минимизация задержки** — данные доставляются практически мгновенно после их появления
3. **Совместимость** — работает везде, где работает HTTP, без необходимости специальных протоколов
4. **Простота реализации** — использует стандартные HTTP-запросы
5. **Обход файрволов и прокси** — не требует специальных портов или протоколов

### Историческая справка

Long Polling стал популярным в середине 2000-х годов как часть технологий, объединённых под названием **Comet**. До появления WebSocket в 2011 году, это была основная техника для создания real-time веб-приложений. Gmail, Facebook Chat и другие крупные сервисы активно использовали Long Polling для доставки уведомлений и сообщений.

---

## Как работает Long Polling

### Базовый механизм

Long Polling работает по следующему принципу:

1. **Клиент отправляет HTTP-запрос** к серверу
2. **Сервер держит соединение открытым**, если нет новых данных
3. **Когда данные появляются**, сервер отправляет ответ
4. **Клиент обрабатывает ответ** и немедленно отправляет новый запрос
5. **Цикл повторяется** бесконечно

### Lifecycle запроса (ASCII-диаграмма)

```
Клиент                                              Сервер
   │                                                   │
   │  ──────── HTTP Request ────────────────────────>  │
   │                                                   │
   │                    (Сервер держит соединение      │
   │                     открытым, ожидая данные)      │
   │                              ...                  │
   │                    (Проходит время: 1с, 5с, 30с)  │
   │                              ...                  │
   │                    (Появились новые данные!)      │
   │                                                   │
   │  <─────── HTTP Response (с данными) ───────────   │
   │                                                   │
   │  (Клиент обрабатывает данные)                     │
   │                                                   │
   │  ──────── Новый HTTP Request ──────────────────>  │
   │                                                   │
   │                    (Цикл повторяется)             │
   │                              ...                  │
```

### Детальная диаграмма с таймаутами

```
    Клиент                                         Сервер
       │                                              │
  t=0  │ ─────────── GET /poll ────────────────────> │
       │                                              │
       │              [Сервер проверяет очередь]      │
       │              [Данных нет - ждём]             │
       │                      ...                     │
       │                                              │
 t=15s │              [Новое сообщение в очереди!]    │
       │                                              │
       │ <────────── 200 OK + JSON data ───────────── │
       │                                              │
       │ [Обработка данных]                           │
       │ [UI обновлён]                                │
       │                                              │
 t=15s │ ─────────── GET /poll ────────────────────> │
       │                                              │
       │              [Проверка очереди]              │
       │              [Данных нет]                    │
       │                      ...                     │
       │                                              │
 t=45s │              [Таймаут сервера - 30с]         │
       │                                              │
       │ <────────── 204 No Content ───────────────── │
       │                                              │
 t=45s │ ─────────── GET /poll ────────────────────> │
       │                      ...                     │
```

### Состояния соединения

```
┌─────────────────────────────────────────────────────────────────┐
│                    Жизненный цикл Long Polling                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│   │  IDLE    │ ──── │ PENDING  │ ──── │ RECEIVED │             │
│   │          │      │          │      │          │             │
│   └──────────┘      └──────────┘      └──────────┘             │
│        │                 │                 │                    │
│        │                 │                 │                    │
│        │                 ▼                 │                    │
│        │           ┌──────────┐            │                    │
│        │           │ TIMEOUT  │            │                    │
│        │           │          │            │                    │
│        │           └──────────┘            │                    │
│        │                 │                 │                    │
│        │                 │                 │                    │
│        └─────────────────┴─────────────────┘                    │
│                          │                                      │
│                          ▼                                      │
│                    Новый запрос                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Примеры кода

### JavaScript клиент (Browser)

#### Базовая реализация

```javascript
class LongPollingClient {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            timeout: 30000,        // Таймаут запроса (30 секунд)
            retryDelay: 1000,      // Задержка перед повторным подключением
            maxRetries: 5,         // Максимум попыток при ошибке
            ...options
        };

        this.isRunning = false;
        this.retryCount = 0;
        this.lastEventId = null;
        this.controller = null;
    }

    // Запуск long polling
    start() {
        if (this.isRunning) {
            console.warn('Long polling уже запущен');
            return;
        }

        this.isRunning = true;
        this.retryCount = 0;
        this.poll();

        console.log('Long polling запущен');
    }

    // Остановка long polling
    stop() {
        this.isRunning = false;

        if (this.controller) {
            this.controller.abort();
            this.controller = null;
        }

        console.log('Long polling остановлен');
    }

    // Основной метод опроса
    async poll() {
        if (!this.isRunning) return;

        this.controller = new AbortController();
        const signal = this.controller.signal;

        try {
            // Формируем URL с параметрами
            const url = new URL(this.url);
            if (this.lastEventId) {
                url.searchParams.set('lastEventId', this.lastEventId);
            }

            const response = await fetch(url, {
                method: 'GET',
                headers: {
                    'Accept': 'application/json',
                    'Cache-Control': 'no-cache'
                },
                signal,
                // Важно: не устанавливаем короткий timeout на fetch
            });

            // Сбрасываем счётчик повторов при успешном ответе
            this.retryCount = 0;

            if (response.status === 200) {
                // Есть новые данные
                const data = await response.json();
                this.handleData(data);

                // Обновляем lastEventId для следующего запроса
                if (data.eventId) {
                    this.lastEventId = data.eventId;
                }
            } else if (response.status === 204) {
                // Таймаут сервера, данных нет
                console.log('Таймаут сервера, переподключаемся...');
            } else if (response.status === 502 || response.status === 503) {
                // Сервер перегружен
                await this.handleError(new Error(`Сервер недоступен: ${response.status}`));
                return;
            }

            // Немедленно начинаем новый запрос
            this.poll();

        } catch (error) {
            if (error.name === 'AbortError') {
                console.log('Запрос отменён');
                return;
            }

            await this.handleError(error);
        }
    }

    // Обработка полученных данных
    handleData(data) {
        console.log('Получены данные:', data);

        // Вызываем callback, если он установлен
        if (this.onMessage) {
            this.onMessage(data);
        }

        // Генерируем кастомное событие
        const event = new CustomEvent('longpoll:message', {
            detail: data
        });
        window.dispatchEvent(event);
    }

    // Обработка ошибок с exponential backoff
    async handleError(error) {
        console.error('Ошибка long polling:', error);

        this.retryCount++;

        if (this.retryCount > this.options.maxRetries) {
            console.error('Превышено максимальное количество попыток');
            this.stop();

            if (this.onError) {
                this.onError(error);
            }
            return;
        }

        // Exponential backoff
        const delay = this.options.retryDelay * Math.pow(2, this.retryCount - 1);
        console.log(`Повторная попытка через ${delay}мс (попытка ${this.retryCount})`);

        await this.sleep(delay);

        if (this.isRunning) {
            this.poll();
        }
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    // Установка обработчиков
    onMessage(callback) {
        this.onMessage = callback;
        return this;
    }

    onError(callback) {
        this.onError = callback;
        return this;
    }
}
```

#### Использование клиента

```javascript
// Создаём экземпляр клиента
const polling = new LongPollingClient('https://api.example.com/poll');

// Устанавливаем обработчик сообщений
polling.onMessage = (data) => {
    console.log('Новое сообщение:', data);

    // Обновляем UI
    if (data.type === 'notification') {
        showNotification(data.message);
    } else if (data.type === 'chat_message') {
        appendMessage(data.content);
    }
};

// Обработчик ошибок
polling.onError = (error) => {
    console.error('Long polling отключён из-за ошибки:', error);
    showReconnectButton();
};

// Запускаем
polling.start();

// Для остановки (например, при выходе со страницы)
window.addEventListener('beforeunload', () => {
    polling.stop();
});
```

### Python сервер (Flask)

```python
from flask import Flask, jsonify, request, Response
from queue import Queue, Empty
from threading import Lock
from datetime import datetime
import uuid
import time

app = Flask(__name__)

# Хранилище для очередей сообщений клиентов
client_queues = {}
client_queues_lock = Lock()

# Глобальный список сообщений (для демонстрации)
messages = []
messages_lock = Lock()

# Таймаут ожидания в секундах
LONG_POLL_TIMEOUT = 30


def get_or_create_client_queue(client_id):
    """Получить или создать очередь для клиента"""
    with client_queues_lock:
        if client_id not in client_queues:
            client_queues[client_id] = Queue()
        return client_queues[client_id]


def broadcast_message(message):
    """Отправить сообщение всем подключённым клиентам"""
    with client_queues_lock:
        for client_id, queue in client_queues.items():
            queue.put(message)


@app.route('/poll', methods=['GET'])
def long_poll():
    """
    Endpoint для long polling.
    Держит соединение открытым до появления новых данных или таймаута.
    """
    # Получаем или генерируем ID клиента
    client_id = request.args.get('client_id')
    if not client_id:
        client_id = str(uuid.uuid4())

    # Получаем последний обработанный event ID
    last_event_id = request.args.get('lastEventId')

    # Получаем очередь клиента
    client_queue = get_or_create_client_queue(client_id)

    try:
        # Ждём сообщение с таймаутом
        message = client_queue.get(timeout=LONG_POLL_TIMEOUT)

        return jsonify({
            'status': 'ok',
            'eventId': str(uuid.uuid4()),
            'clientId': client_id,
            'data': message,
            'timestamp': datetime.utcnow().isoformat()
        })

    except Empty:
        # Таймаут - нет новых данных
        return Response(status=204)

    except Exception as e:
        return jsonify({
            'status': 'error',
            'message': str(e)
        }), 500


@app.route('/send', methods=['POST'])
def send_message():
    """Endpoint для отправки сообщений (для тестирования)"""
    data = request.get_json()

    if not data or 'message' not in data:
        return jsonify({'error': 'Message required'}), 400

    message = {
        'id': str(uuid.uuid4()),
        'content': data['message'],
        'type': data.get('type', 'text'),
        'timestamp': datetime.utcnow().isoformat()
    }

    # Сохраняем сообщение
    with messages_lock:
        messages.append(message)

    # Рассылаем всем клиентам
    broadcast_message(message)

    return jsonify({
        'status': 'ok',
        'message': message
    })


@app.route('/cleanup', methods=['POST'])
def cleanup_clients():
    """Очистка неактивных клиентов"""
    with client_queues_lock:
        # В реальном приложении здесь была бы логика
        # определения неактивных клиентов
        count = len(client_queues)
        client_queues.clear()

    return jsonify({
        'status': 'ok',
        'cleaned': count
    })


if __name__ == '__main__':
    app.run(debug=True, threaded=True, port=5000)
```

### Python сервер (FastAPI) — асинхронная версия

```python
from fastapi import FastAPI, Query, HTTPException
from fastapi.responses import JSONResponse, Response
from pydantic import BaseModel
from typing import Optional, Dict, List
from datetime import datetime
from asyncio import Queue, wait_for, TimeoutError as AsyncTimeoutError
import uuid

app = FastAPI(title="Long Polling Server")

# Хранилище очередей клиентов
client_queues: Dict[str, Queue] = {}

# Таймаут в секундах
LONG_POLL_TIMEOUT = 30


class Message(BaseModel):
    content: str
    type: str = "text"


class MessageResponse(BaseModel):
    id: str
    content: str
    type: str
    timestamp: str


async def get_or_create_queue(client_id: str) -> Queue:
    """Получить или создать очередь для клиента"""
    if client_id not in client_queues:
        client_queues[client_id] = Queue()
    return client_queues[client_id]


async def broadcast_message(message: dict):
    """Отправить сообщение всем подключённым клиентам"""
    for queue in client_queues.values():
        await queue.put(message)


@app.get("/poll")
async def long_poll(
    client_id: Optional[str] = Query(None, description="Идентификатор клиента"),
    last_event_id: Optional[str] = Query(None, alias="lastEventId")
):
    """
    Long Polling endpoint.

    Держит соединение открытым до:
    - Появления новых данных
    - Истечения таймаута (30 секунд)

    Returns:
        - 200 + JSON: если есть данные
        - 204: если таймаут без данных
    """
    # Генерируем client_id если не передан
    if not client_id:
        client_id = str(uuid.uuid4())

    # Получаем очередь
    queue = await get_or_create_queue(client_id)

    try:
        # Асинхронно ждём данные с таймаутом
        message = await wait_for(
            queue.get(),
            timeout=LONG_POLL_TIMEOUT
        )

        return JSONResponse(content={
            "status": "ok",
            "eventId": str(uuid.uuid4()),
            "clientId": client_id,
            "data": message,
            "timestamp": datetime.utcnow().isoformat()
        })

    except AsyncTimeoutError:
        # Таймаут - возвращаем 204 No Content
        return Response(status_code=204)


@app.post("/send")
async def send_message(message: Message):
    """
    Отправить сообщение всем подключённым клиентам.

    Используется для тестирования long polling.
    """
    msg = {
        "id": str(uuid.uuid4()),
        "content": message.content,
        "type": message.type,
        "timestamp": datetime.utcnow().isoformat()
    }

    # Broadcast всем клиентам
    await broadcast_message(msg)

    return {
        "status": "ok",
        "message": msg,
        "clients_notified": len(client_queues)
    }


@app.get("/clients")
async def list_clients():
    """Список активных клиентов"""
    return {
        "count": len(client_queues),
        "client_ids": list(client_queues.keys())
    }


@app.delete("/clients/{client_id}")
async def remove_client(client_id: str):
    """Удалить клиента"""
    if client_id in client_queues:
        del client_queues[client_id]
        return {"status": "ok", "message": f"Client {client_id} removed"}

    raise HTTPException(status_code=404, detail="Client not found")


# Middleware для CORS (если нужно)
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Пример чат-приложения

#### Сервер (FastAPI)

```python
from fastapi import FastAPI, WebSocket, Query
from fastapi.responses import HTMLResponse, JSONResponse, Response
from typing import Dict, List, Optional
from asyncio import Queue, wait_for, TimeoutError
from datetime import datetime
from pydantic import BaseModel
import uuid

app = FastAPI()

# Хранилище
chat_rooms: Dict[str, Dict[str, Queue]] = {}  # room_id -> {client_id -> queue}
chat_history: Dict[str, List[dict]] = {}       # room_id -> [messages]


class ChatMessage(BaseModel):
    room_id: str
    username: str
    content: str


async def get_room_queue(room_id: str, client_id: str) -> Queue:
    """Получить очередь клиента в комнате"""
    if room_id not in chat_rooms:
        chat_rooms[room_id] = {}
        chat_history[room_id] = []

    if client_id not in chat_rooms[room_id]:
        chat_rooms[room_id][client_id] = Queue()

    return chat_rooms[room_id][client_id]


async def broadcast_to_room(room_id: str, message: dict, exclude_client: str = None):
    """Отправить сообщение всем в комнате"""
    if room_id not in chat_rooms:
        return

    for client_id, queue in chat_rooms[room_id].items():
        if client_id != exclude_client:
            await queue.put(message)


@app.get("/chat/{room_id}/poll")
async def poll_messages(
    room_id: str,
    client_id: str = Query(...),
    last_message_id: Optional[str] = Query(None)
):
    """Long polling для получения сообщений чата"""
    queue = await get_room_queue(room_id, client_id)

    try:
        message = await wait_for(queue.get(), timeout=30)

        return JSONResponse({
            "status": "message",
            "data": message
        })

    except TimeoutError:
        return Response(status_code=204)


@app.post("/chat/{room_id}/send")
async def send_chat_message(room_id: str, message: ChatMessage):
    """Отправить сообщение в чат"""
    msg = {
        "id": str(uuid.uuid4()),
        "room_id": room_id,
        "username": message.username,
        "content": message.content,
        "timestamp": datetime.utcnow().isoformat()
    }

    # Сохраняем в историю
    if room_id not in chat_history:
        chat_history[room_id] = []
    chat_history[room_id].append(msg)

    # Ограничиваем историю последними 100 сообщениями
    chat_history[room_id] = chat_history[room_id][-100:]

    # Рассылаем всем в комнате
    await broadcast_to_room(room_id, msg)

    return {"status": "ok", "message": msg}


@app.get("/chat/{room_id}/history")
async def get_history(room_id: str, limit: int = 50):
    """Получить историю сообщений"""
    history = chat_history.get(room_id, [])
    return {"messages": history[-limit:]}


@app.post("/chat/{room_id}/join")
async def join_room(room_id: str, client_id: str = Query(...), username: str = Query(...)):
    """Присоединиться к комнате"""
    await get_room_queue(room_id, client_id)

    # Уведомляем остальных
    join_message = {
        "id": str(uuid.uuid4()),
        "type": "system",
        "content": f"{username} присоединился к чату",
        "timestamp": datetime.utcnow().isoformat()
    }
    await broadcast_to_room(room_id, join_message, exclude_client=client_id)

    return {"status": "ok", "room_id": room_id}


@app.post("/chat/{room_id}/leave")
async def leave_room(room_id: str, client_id: str = Query(...), username: str = Query(...)):
    """Покинуть комнату"""
    if room_id in chat_rooms and client_id in chat_rooms[room_id]:
        del chat_rooms[room_id][client_id]

        # Уведомляем остальных
        leave_message = {
            "id": str(uuid.uuid4()),
            "type": "system",
            "content": f"{username} покинул чат",
            "timestamp": datetime.utcnow().isoformat()
        }
        await broadcast_to_room(room_id, leave_message)

    return {"status": "ok"}
```

#### JavaScript клиент для чата

```javascript
class ChatClient {
    constructor(roomId, username) {
        this.roomId = roomId;
        this.username = username;
        this.clientId = this.generateClientId();
        this.baseUrl = '/api';
        this.isConnected = false;
        this.messageCallbacks = [];
    }

    generateClientId() {
        return 'client_' + Math.random().toString(36).substr(2, 9);
    }

    async connect() {
        // Присоединяемся к комнате
        const response = await fetch(
            `${this.baseUrl}/chat/${this.roomId}/join?client_id=${this.clientId}&username=${this.username}`,
            { method: 'POST' }
        );

        if (response.ok) {
            this.isConnected = true;

            // Загружаем историю
            await this.loadHistory();

            // Начинаем long polling
            this.poll();
        }
    }

    async loadHistory() {
        const response = await fetch(
            `${this.baseUrl}/chat/${this.roomId}/history?limit=50`
        );

        if (response.ok) {
            const data = await response.json();
            data.messages.forEach(msg => this.notifyMessage(msg));
        }
    }

    async poll() {
        if (!this.isConnected) return;

        try {
            const response = await fetch(
                `${this.baseUrl}/chat/${this.roomId}/poll?client_id=${this.clientId}`
            );

            if (response.status === 200) {
                const data = await response.json();
                this.notifyMessage(data.data);
            }

            // Продолжаем polling
            this.poll();

        } catch (error) {
            console.error('Polling error:', error);

            // Переподключение через 2 секунды
            setTimeout(() => this.poll(), 2000);
        }
    }

    async sendMessage(content) {
        const response = await fetch(
            `${this.baseUrl}/chat/${this.roomId}/send`,
            {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    room_id: this.roomId,
                    username: this.username,
                    content: content
                })
            }
        );

        return response.ok;
    }

    onMessage(callback) {
        this.messageCallbacks.push(callback);
    }

    notifyMessage(message) {
        this.messageCallbacks.forEach(cb => cb(message));
    }

    async disconnect() {
        this.isConnected = false;

        await fetch(
            `${this.baseUrl}/chat/${this.roomId}/leave?client_id=${this.clientId}&username=${this.username}`,
            { method: 'POST' }
        );
    }
}

// Использование
const chat = new ChatClient('room123', 'Иван');

chat.onMessage((message) => {
    console.log(`[${message.username}]: ${message.content}`);
    // Обновляем UI
    appendMessageToChat(message);
});

// Подключаемся
await chat.connect();

// Отправляем сообщение
await chat.sendMessage('Привет всем!');

// При закрытии страницы
window.addEventListener('beforeunload', () => {
    chat.disconnect();
});
```

---

## Преимущества и недостатки

### Сравнительная таблица

| Аспект | Преимущества | Недостатки |
|--------|-------------|------------|
| **Совместимость** | Работает везде, где есть HTTP | — |
| **Реализация** | Простая — только HTTP запросы | Сложнее, чем short polling |
| **Задержка** | Минимальная — данные приходят сразу | Небольшая задержка на переподключение |
| **Нагрузка сервера** | Меньше запросов, чем short polling | Держит открытые соединения |
| **Масштабирование** | — | Каждый клиент = открытое соединение |
| **Прокси/Firewall** | Проходит через любые прокси | Некоторые прокси закрывают долгие соединения |
| **Двунаправленность** | — | Только сервер → клиент |
| **Батарея (мобильные)** | — | Держит радио активным |

### Детальный анализ

#### Преимущества

1. **Универсальная совместимость**
   - Работает во всех браузерах
   - Проходит через корпоративные файрволы
   - Не требует специальных протоколов

2. **Простота отладки**
   - Обычные HTTP запросы видны в DevTools
   - Легко логировать и мониторить
   - Стандартные инструменты работают

3. **Надёжность**
   - Автоматическое переподключение при обрыве
   - Можно реализовать гарантированную доставку
   - Поддержка last-event-id

4. **Меньшая нагрузка**
   - Один запрос вместо множества (сравнивая с short polling)
   - Меньше трафика
   - Меньше парсинга заголовков

#### Недостатки

1. **Ресурсы сервера**
   - Каждый клиент держит соединение
   - Ограничение на количество соединений
   - Потребляет память

2. **Масштабирование**
   - Сложнее масштабировать, чем stateless API
   - Нужен sticky sessions или pub/sub
   - Балансировка нагрузки сложнее

3. **Односторонняя связь**
   - Клиент не может отправить данные через polling-соединение
   - Нужен отдельный endpoint для отправки

4. **Latency на переподключение**
   - После получения данных нужен новый запрос
   - Небольшая задержка между ответом и новым запросом

---

## Когда использовать Long Polling

### Идеальные сценарии

1. **WebSocket недоступен**
   - Корпоративные сети с ограничениями
   - Старые браузеры
   - Прокси, блокирующие WebSocket

2. **Простые real-time приложения**
   - Уведомления
   - Обновления статуса
   - Ленты новостей

3. **Редкие обновления**
   - Изменения происходят раз в несколько секунд/минут
   - Не критична миллисекундная задержка

4. **Fallback механизм**
   - Как запасной вариант при сбое WebSocket
   - Автоматическое переключение

### Когда НЕ использовать

1. **Высокочастотные обновления**
   - Игры в реальном времени
   - Трейдинг
   - Совместное редактирование

2. **Двусторонняя связь**
   - Видеозвонки
   - Чаты с печатью "пишет..."
   - Интерактивные приложения

3. **Большое количество клиентов**
   - Миллионы одновременных подключений
   - Без возможности масштабирования

### Пример fallback логики

```javascript
class RealtimeConnection {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.polling = null;
    }

    connect() {
        // Пробуем WebSocket
        try {
            this.ws = new WebSocket(this.url.replace('http', 'ws') + '/ws');

            this.ws.onopen = () => {
                console.log('WebSocket подключён');
            };

            this.ws.onmessage = (event) => {
                this.handleMessage(JSON.parse(event.data));
            };

            this.ws.onerror = () => {
                console.log('WebSocket недоступен, переключаемся на Long Polling');
                this.fallbackToLongPolling();
            };

            this.ws.onclose = () => {
                if (!this.polling) {
                    this.fallbackToLongPolling();
                }
            };

        } catch (error) {
            this.fallbackToLongPolling();
        }
    }

    fallbackToLongPolling() {
        if (this.ws) {
            this.ws.close();
            this.ws = null;
        }

        this.polling = new LongPollingClient(this.url + '/poll');
        this.polling.onMessage = (data) => this.handleMessage(data);
        this.polling.start();
    }

    handleMessage(data) {
        // Общая обработка сообщений
        console.log('Получено:', data);
    }

    send(data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        } else {
            // Для Long Polling отправляем через обычный POST
            fetch(this.url + '/send', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(data)
            });
        }
    }
}
```

---

## Сравнение с Short Polling и WebSocket

### Сравнительная таблица технологий

| Характеристика | Short Polling | Long Polling | WebSocket |
|---------------|---------------|--------------|-----------|
| **Протокол** | HTTP | HTTP | WS/WSS |
| **Соединение** | Новое каждый раз | Держится до ответа | Постоянное |
| **Направление** | Клиент → Сервер | Сервер → Клиент | Двустороннее |
| **Латентность** | Высокая (интервал) | Низкая | Очень низкая |
| **Нагрузка сервера** | Высокая (много запросов) | Средняя | Низкая |
| **Нагрузка сети** | Высокая | Средняя | Низкая |
| **Сложность** | Простая | Средняя | Высокая |
| **Совместимость** | 100% | 100% | ~97% |
| **Firewall/Proxy** | Всегда проходит | Почти всегда | Иногда блокируется |
| **Масштабируемость** | Хорошая | Средняя | Хорошая |

### Визуальное сравнение

```
SHORT POLLING (интервал 1 сек)
═══════════════════════════════════════════════════════════════════

Клиент: ─●─────●─────●─────●─────●─────●─────●─────●─────●─────●──>
         ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓     ↓
Сервер: ─◯─────◯─────◯─────●─────◯─────◯─────◯─────●─────◯─────◯──>
                           ↑                       ↑
                         Данные                  Данные
                      (задержка до 1с)       (задержка до 1с)


LONG POLLING
═══════════════════════════════════════════════════════════════════

Клиент: ─●─────────────────────────●───────────────────────●──────>
         │                         │                       │
         ▼   (ожидание данных)     ▼   (ожидание)          ▼
Сервер: ─────────────────◯─────────────────────────◯───────────────>
         [   держит    ]│↑         [   держит    ]│↑
         [ соединение  ]│ мгновенно[ соединение  ]│ мгновенно
                       Данные                    Данные


WEBSOCKET
═══════════════════════════════════════════════════════════════════

         Постоянное двустороннее соединение
         ════════════════════════════════════════════════════>
              ↓              ↓     ↑     ↓         ↓
Клиент: ─────●──────────────────────────●─────────────────────────>
                             │     │           │
Сервер: ─────────────────────●─────●───────────●──────────────────>
                             ↑     ↑           ↑
                         Данные от сервера (мгновенно)
```

### Пример: что выбрать?

```javascript
// Функция выбора технологии
function chooseRealtimeTechnology(requirements) {
    const {
        updateFrequency,      // updates per second
        bidirectional,        // нужна двусторонняя связь?
        browserSupport,       // минимальная версия браузера
        corporateNetwork,     // корпоративная сеть с ограничениями?
        expectedClients       // ожидаемое количество клиентов
    } = requirements;

    // Высокая частота обновлений или двусторонняя связь
    if (updateFrequency > 1 || bidirectional) {
        if (corporateNetwork) {
            console.log('Рекомендуется: WebSocket с fallback на Long Polling');
            return 'websocket-with-fallback';
        }
        return 'websocket';
    }

    // Редкие обновления
    if (updateFrequency < 0.1) { // менее 1 раза в 10 секунд
        if (corporateNetwork) {
            return 'long-polling';
        }
        // Для редких обновлений short polling может быть проще
        return 'short-polling';
    }

    // Средняя частота, простые требования
    if (!bidirectional && !corporateNetwork) {
        return 'long-polling';
    }

    // По умолчанию: Long Polling как универсальное решение
    return 'long-polling';
}

// Примеры использования
console.log(chooseRealtimeTechnology({
    updateFrequency: 0.5,      // раз в 2 секунды
    bidirectional: false,
    corporateNetwork: true
})); // 'long-polling'

console.log(chooseRealtimeTechnology({
    updateFrequency: 10,       // 10 раз в секунду
    bidirectional: true,
    corporateNetwork: false
})); // 'websocket'
```

---

## Оптимизация Long Polling

### Серверная оптимизация

```python
# Python (FastAPI) с оптимизациями
from fastapi import FastAPI
from asyncio import Queue, wait_for, TimeoutError, create_task
from contextlib import asynccontextmanager
import asyncio

app = FastAPI()

# Пул соединений с ограничением
MAX_CONNECTIONS = 10000
connection_semaphore = asyncio.Semaphore(MAX_CONNECTIONS)


@asynccontextmanager
async def limit_connections():
    """Ограничиваем количество одновременных соединений"""
    async with connection_semaphore:
        yield


@app.get("/poll")
async def optimized_poll(client_id: str):
    async with limit_connections():
        queue = await get_or_create_queue(client_id)

        try:
            # Batch: собираем все доступные сообщения
            messages = []

            try:
                # Первое сообщение - ждём
                msg = await wait_for(queue.get(), timeout=30)
                messages.append(msg)

                # Дополнительные - берём без ожидания
                while not queue.empty() and len(messages) < 10:
                    messages.append(queue.get_nowait())

            except TimeoutError:
                pass

            if messages:
                return {"status": "ok", "messages": messages}
            else:
                return Response(status_code=204)

        except Exception as e:
            logger.error(f"Poll error: {e}")
            raise
```

### Клиентская оптимизация

```javascript
class OptimizedLongPolling {
    constructor(url) {
        this.url = url;
        this.buffer = [];
        this.flushInterval = null;
    }

    start() {
        // Batch обработка сообщений
        this.flushInterval = setInterval(() => {
            if (this.buffer.length > 0) {
                this.processBuffer();
            }
        }, 100); // Обрабатываем каждые 100мс

        this.poll();
    }

    async poll() {
        try {
            const response = await fetch(this.url);

            if (response.ok) {
                const data = await response.json();

                // Буферизуем сообщения
                if (Array.isArray(data.messages)) {
                    this.buffer.push(...data.messages);
                }
            }

            // Немедленно новый запрос
            requestAnimationFrame(() => this.poll());

        } catch (error) {
            // Exponential backoff при ошибках
            await this.backoff();
            this.poll();
        }
    }

    processBuffer() {
        const messages = this.buffer.splice(0, this.buffer.length);

        // Batch DOM update
        requestAnimationFrame(() => {
            messages.forEach(msg => this.renderMessage(msg));
        });
    }
}
```

---

## Заключение

Long Polling остаётся актуальной технологией для создания real-time приложений, особенно в ситуациях, когда:

- WebSocket недоступен или блокируется
- Требуется максимальная совместимость
- Приложение не требует высокой частоты обновлений
- Нужен надёжный fallback механизм

При правильной реализации Long Polling обеспечивает хороший баланс между простотой разработки и качеством пользовательского опыта.

### Ключевые выводы

1. **Long Polling — это эволюция** Short Polling, уменьшающая нагрузку и задержку
2. **Используйте таймауты** на сервере (30-60 секунд) для предотвращения зависших соединений
3. **Реализуйте reconnect логику** с exponential backoff
4. **Рассмотрите WebSocket** для высокочастотных обновлений или двусторонней связи
5. **Комбинируйте технологии** — WebSocket с fallback на Long Polling

---

[prev: 02-server-sent-events](./02-server-sent-events.md) | [next: 04-short-polling](./04-short-polling.md)
