# WebSockets

## Введение

**WebSocket** — это протокол связи поверх TCP-соединения, обеспечивающий полнодуплексный (двунаправленный) обмен данными между клиентом и сервером в режиме реального времени. В отличие от традиционного HTTP, где клиент инициирует каждый запрос, WebSocket позволяет обеим сторонам отправлять сообщения в любой момент после установления соединения.

WebSocket был стандартизирован IETF как RFC 6455 в 2011 году и является ключевой технологией для создания интерактивных веб-приложений.

---

## Принцип работы

### Установление соединения (Handshake)

WebSocket-соединение начинается с HTTP-запроса, который затем "апгрейдится" до WebSocket:

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

Сервер отвечает:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

После успешного handshake TCP-соединение остаётся открытым, и обе стороны могут обмениваться данными.

### Структура фрейма

WebSocket использует фреймы для передачи данных:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### Типы опкодов

| Opcode | Описание |
|--------|----------|
| 0x0 | Continuation frame |
| 0x1 | Text frame (UTF-8) |
| 0x2 | Binary frame |
| 0x8 | Close connection |
| 0x9 | Ping |
| 0xA | Pong |

---

## Сравнение с альтернативами

### WebSocket vs HTTP Polling

| Характеристика | WebSocket | HTTP Polling |
|---------------|-----------|--------------|
| Задержка | Минимальная | Зависит от интервала |
| Нагрузка на сервер | Низкая | Высокая (много запросов) |
| Двунаправленность | Да | Нет (только клиент инициирует) |
| Сложность реализации | Средняя | Низкая |
| Поддержка прокси | Может быть проблемой | Отличная |

### WebSocket vs Long Polling

| Характеристика | WebSocket | Long Polling |
|---------------|-----------|--------------|
| Постоянное соединение | Да | Нет (переподключение) |
| Overhead | Минимальный | HTTP заголовки каждый раз |
| Масштабируемость | Лучше | Хуже |

### WebSocket vs Server-Sent Events (SSE)

| Характеристика | WebSocket | SSE |
|---------------|-----------|-----|
| Направление | Двунаправленное | Только сервер -> клиент |
| Протокол | Бинарный + текст | Только текст |
| Автопереподключение | Нужно реализовать | Встроено |
| Поддержка браузеров | Отличная | Отличная (кроме IE) |

---

## Практические примеры кода

### Python (сервер с использованием websockets)

```python
import asyncio
import websockets
import json

# Хранилище подключённых клиентов
connected_clients = set()

async def handler(websocket, path):
    """Обработчик WebSocket соединений"""
    # Регистрируем нового клиента
    connected_clients.add(websocket)
    print(f"Клиент подключился. Всего клиентов: {len(connected_clients)}")

    try:
        async for message in websocket:
            # Парсим входящее сообщение
            data = json.loads(message)
            print(f"Получено: {data}")

            # Рассылаем сообщение всем подключённым клиентам
            broadcast_message = json.dumps({
                "type": "broadcast",
                "sender": data.get("username", "Anonymous"),
                "content": data.get("message", "")
            })

            await asyncio.gather(
                *[client.send(broadcast_message) for client in connected_clients]
            )
    except websockets.exceptions.ConnectionClosed:
        print("Клиент отключился")
    finally:
        connected_clients.remove(websocket)

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        print("WebSocket сервер запущен на ws://localhost:8765")
        await asyncio.Future()  # Работаем бесконечно

if __name__ == "__main__":
    asyncio.run(main())
```

### Python (клиент)

```python
import asyncio
import websockets
import json

async def client():
    uri = "ws://localhost:8765"

    async with websockets.connect(uri) as websocket:
        # Отправляем приветственное сообщение
        await websocket.send(json.dumps({
            "username": "User1",
            "message": "Привет, мир!"
        }))

        # Слушаем ответы от сервера
        async for message in websocket:
            data = json.loads(message)
            print(f"[{data['sender']}]: {data['content']}")

if __name__ == "__main__":
    asyncio.run(client())
```

### Python (FastAPI с WebSocket)

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class ConnectionManager:
    """Менеджер WebSocket соединений"""

    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    await manager.broadcast(f"Пользователь {client_id} присоединился")

    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"{client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Пользователь {client_id} покинул чат")
```

### JavaScript (клиент в браузере)

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.socket = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.reconnectDelay = 1000;
    }

    connect() {
        this.socket = new WebSocket(this.url);

        this.socket.onopen = () => {
            console.log('WebSocket соединение установлено');
            this.reconnectAttempts = 0;
            this.onConnected();
        };

        this.socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.onMessage(data);
        };

        this.socket.onclose = (event) => {
            console.log('WebSocket соединение закрыто:', event.code, event.reason);
            this.handleReconnect();
        };

        this.socket.onerror = (error) => {
            console.error('WebSocket ошибка:', error);
        };
    }

    handleReconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            console.log(`Попытка переподключения ${this.reconnectAttempts}/${this.maxReconnectAttempts}`);

            setTimeout(() => {
                this.connect();
            }, this.reconnectDelay * this.reconnectAttempts);
        } else {
            console.error('Превышено максимальное количество попыток переподключения');
        }
    }

    send(data) {
        if (this.socket && this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(JSON.stringify(data));
        } else {
            console.error('WebSocket не подключён');
        }
    }

    close() {
        if (this.socket) {
            this.socket.close();
        }
    }

    // Переопределяемые методы
    onConnected() {}
    onMessage(data) {}
}

// Использование
const client = new WebSocketClient('ws://localhost:8765');

client.onConnected = () => {
    client.send({
        username: 'WebUser',
        message: 'Привет с браузера!'
    });
};

client.onMessage = (data) => {
    console.log(`[${data.sender}]: ${data.content}`);
    // Добавляем сообщение в DOM
    const chatBox = document.getElementById('chat');
    chatBox.innerHTML += `<p><strong>${data.sender}:</strong> ${data.content}</p>`;
};

client.connect();
```

### JavaScript (Node.js сервер с ws)

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8765 });

// Хранилище клиентов с метаданными
const clients = new Map();

wss.on('connection', (ws, req) => {
    const clientId = Date.now().toString();
    clients.set(ws, { id: clientId, joinedAt: new Date() });

    console.log(`Клиент ${clientId} подключился`);

    // Отправляем приветствие
    ws.send(JSON.stringify({
        type: 'welcome',
        message: `Добро пожаловать! Ваш ID: ${clientId}`
    }));

    // Обработка входящих сообщений
    ws.on('message', (message) => {
        const data = JSON.parse(message);
        console.log(`Получено от ${clientId}:`, data);

        // Рассылка всем клиентам
        wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(JSON.stringify({
                    type: 'message',
                    from: clientId,
                    content: data.message,
                    timestamp: new Date().toISOString()
                }));
            }
        });
    });

    // Heartbeat (ping/pong)
    ws.isAlive = true;
    ws.on('pong', () => {
        ws.isAlive = true;
    });

    ws.on('close', () => {
        clients.delete(ws);
        console.log(`Клиент ${clientId} отключился`);
    });
});

// Проверка живых соединений каждые 30 секунд
const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (!ws.isAlive) {
            return ws.terminate();
        }
        ws.isAlive = false;
        ws.ping();
    });
}, 30000);

wss.on('close', () => {
    clearInterval(interval);
});

console.log('WebSocket сервер запущен на ws://localhost:8765');
```

---

## Примеры использования в реальных приложениях

### 1. Чат-приложения
- Мгновенный обмен сообщениями
- Индикаторы "печатает..."
- Статус онлайн/офлайн

### 2. Онлайн-игры
- Синхронизация состояния игры
- Мультиплеер в реальном времени
- Минимальная задержка (latency)

### 3. Финансовые приложения
- Биржевые котировки в реальном времени
- Торговые платформы
- Криптовалютные биржи

### 4. Коллаборативные инструменты
- Google Docs-подобные редакторы
- Совместные доски (Miro, Figma)
- Синхронизация курсоров

### 5. IoT и мониторинг
- Дашборды с метриками
- Мониторинг серверов
- Умный дом

### 6. Уведомления
- Push-уведомления в веб-приложениях
- Системы оповещения
- Обновления в социальных сетях

---

## Best Practices

### 1. Heartbeat (Ping/Pong)
```python
# Сервер отправляет ping каждые 30 секунд
async def heartbeat(websocket):
    while True:
        try:
            await websocket.ping()
            await asyncio.sleep(30)
        except websockets.exceptions.ConnectionClosed:
            break
```

### 2. Graceful Reconnection
```javascript
// Экспоненциальный backoff при переподключении
function getReconnectDelay(attempt) {
    const baseDelay = 1000;
    const maxDelay = 30000;
    const delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
    return delay + Math.random() * 1000; // Добавляем jitter
}
```

### 3. Аутентификация
```python
# Аутентификация через query параметры
async def handler(websocket, path):
    token = parse_qs(urlparse(path).query).get('token', [None])[0]
    if not validate_token(token):
        await websocket.close(1008, "Invalid token")
        return
```

### 4. Обработка больших сообщений
```python
# Ограничение размера сообщений
async with websockets.serve(
    handler,
    "localhost",
    8765,
    max_size=1024 * 1024  # 1 MB
):
    await asyncio.Future()
```

### 5. Использование комнат (rooms)
```python
rooms = defaultdict(set)

async def join_room(websocket, room_name):
    rooms[room_name].add(websocket)

async def broadcast_to_room(room_name, message):
    for client in rooms[room_name]:
        await client.send(message)
```

---

## Типичные ошибки

### 1. Отсутствие обработки отключений
```python
# Плохо - не обрабатывается разрыв соединения
async for message in websocket:
    process(message)

# Хорошо
try:
    async for message in websocket:
        process(message)
except websockets.exceptions.ConnectionClosed:
    cleanup()
finally:
    remove_client(websocket)
```

### 2. Утечки памяти
```python
# Плохо - клиенты не удаляются
connected_clients.add(websocket)

# Хорошо - обязательно удаляем при отключении
try:
    connected_clients.add(websocket)
    # ...
finally:
    connected_clients.discard(websocket)
```

### 3. Блокирующие операции
```python
# Плохо - блокирует event loop
async def handler(websocket):
    data = heavy_computation()  # Синхронная операция
    await websocket.send(data)

# Хорошо - выносим в executor
async def handler(websocket):
    loop = asyncio.get_event_loop()
    data = await loop.run_in_executor(None, heavy_computation)
    await websocket.send(data)
```

### 4. Отсутствие валидации данных
```python
# Плохо - нет валидации
async def handler(websocket):
    data = json.loads(await websocket.recv())
    execute_command(data['command'])  # Опасно!

# Хорошо - валидация входных данных
async def handler(websocket):
    try:
        data = json.loads(await websocket.recv())
        if 'command' not in data:
            raise ValueError("Missing command")
        if data['command'] not in ALLOWED_COMMANDS:
            raise ValueError("Invalid command")
        execute_command(data['command'])
    except (json.JSONDecodeError, ValueError) as e:
        await websocket.send(json.dumps({"error": str(e)}))
```

### 5. Игнорирование масштабируемости
```python
# Проблема: при нескольких инстансах сервера клиенты
# подключённые к разным инстансам не видят сообщения друг друга

# Решение: использовать Redis Pub/Sub или аналог
import aioredis

async def subscribe_to_redis():
    redis = await aioredis.create_redis_pool('redis://localhost')
    channel = (await redis.subscribe('chat'))[0]
    async for message in channel.iter():
        await broadcast_to_local_clients(message)
```

---

## Безопасность

### 1. Используйте WSS (WebSocket Secure)
```javascript
// Всегда используйте wss:// в production
const socket = new WebSocket('wss://example.com/socket');
```

### 2. Origin проверка
```python
async def handler(websocket, path):
    origin = websocket.request_headers.get('Origin')
    if origin not in ALLOWED_ORIGINS:
        await websocket.close(1008, "Origin not allowed")
        return
```

### 3. Rate limiting
```python
from collections import defaultdict
import time

message_counts = defaultdict(list)
MAX_MESSAGES_PER_MINUTE = 60

async def rate_limit_check(websocket):
    client_ip = websocket.remote_address[0]
    now = time.time()

    # Удаляем старые записи
    message_counts[client_ip] = [
        t for t in message_counts[client_ip]
        if now - t < 60
    ]

    if len(message_counts[client_ip]) >= MAX_MESSAGES_PER_MINUTE:
        await websocket.close(1008, "Rate limit exceeded")
        return False

    message_counts[client_ip].append(now)
    return True
```

---

## Заключение

WebSocket — мощный протокол для создания real-time приложений. Он обеспечивает низкую задержку и двунаправленную связь, что делает его идеальным выбором для чатов, игр, систем мониторинга и других интерактивных приложений. Однако важно правильно обрабатывать отключения, реализовывать heartbeat-механизмы и заботиться о безопасности соединений.
