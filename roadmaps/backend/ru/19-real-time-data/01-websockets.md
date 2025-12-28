# WebSockets (Веб-сокеты)

[prev: 03-next-steps](../18-message-brokers/02-kafka/12-stream-processing/03-next-steps.md) | [next: 02-server-sent-events](./02-server-sent-events.md)

---

## Введение

**WebSocket** — это протокол связи, обеспечивающий полнодуплексное (двунаправленное) соединение между клиентом и сервером через одно TCP-соединение. В отличие от традиционного HTTP, где клиент всегда инициирует запрос, WebSocket позволяет серверу самостоятельно отправлять данные клиенту в любой момент.

### Зачем нужны WebSockets?

В традиционной модели HTTP взаимодействие происходит по принципу "запрос-ответ":
1. Клиент отправляет запрос
2. Сервер обрабатывает и возвращает ответ
3. Соединение закрывается

Это отлично работает для статических веб-страниц, но создаёт проблемы для приложений реального времени:

- **Чаты**: пользователи должны мгновенно получать новые сообщения
- **Онлайн-игры**: игроки должны видеть действия других игроков без задержек
- **Биржевые котировки**: трейдерам нужны актуальные цены в реальном времени
- **Уведомления**: push-уведомления должны приходить мгновенно
- **Совместное редактирование**: изменения одного пользователя должны сразу отображаться у других

### История создания

WebSocket был стандартизирован в 2011 году как часть спецификации RFC 6455. Протокол был разработан для решения проблем, связанных с полудуплексной природой HTTP, и является частью инициативы HTML5.

---

## Как работает WebSocket

### Установление соединения (Handshake)

WebSocket соединение начинается как обычный HTTP-запрос, который затем "улучшается" (upgrade) до WebSocket протокола. Этот процесс называется **handshake** (рукопожатие).

#### Запрос клиента

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
```

Ключевые заголовки:
- `Upgrade: websocket` — указывает на желание переключиться на WebSocket
- `Connection: Upgrade` — подтверждает намерение изменить протокол
- `Sec-WebSocket-Key` — случайный ключ в Base64 для проверки подлинности
- `Sec-WebSocket-Version: 13` — версия протокола WebSocket

#### Ответ сервера

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

Код `101 Switching Protocols` означает успешное переключение на WebSocket. Заголовок `Sec-WebSocket-Accept` содержит хеш, вычисленный из клиентского ключа.

### Диаграмма установления соединения

```
┌─────────────┐                              ┌─────────────┐
│   Клиент    │                              │   Сервер    │
└──────┬──────┘                              └──────┬──────┘
       │                                            │
       │  HTTP GET /chat                            │
       │  Upgrade: websocket                        │
       │  Connection: Upgrade                       │
       │  Sec-WebSocket-Key: xxx                    │
       │ ─────────────────────────────────────────> │
       │                                            │
       │            HTTP/1.1 101 Switching          │
       │            Upgrade: websocket              │
       │            Sec-WebSocket-Accept: yyy       │
       │ <───────────────────────────────────────── │
       │                                            │
       │  ═══════════════════════════════════════   │
       │     WebSocket соединение установлено       │
       │  ═══════════════════════════════════════   │
       │                                            │
       │  Двунаправленный обмен сообщениями         │
       │ <════════════════════════════════════════> │
       │                                            │
```

### Структура фрейма WebSocket

После установления соединения данные передаются в виде **фреймов** (frames). Каждый фрейм имеет следующую структуру:

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| опкод |M| Длина       |    Расширенная длина          |
 |I|S|S|S|  (4)  |A| payload (7) |    payload (16/64 бит)        |
 |N|V|V|V|       |S|             |    (если payload len == 126/127)
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Расширенная длина payload (продолжение, если 64 бит)      |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |    Ключ маскирования (если    |
 |                               |    MASK установлен)           |
 +-------------------------------+-------------------------------+
 |    Ключ маскирования          |    Данные payload             |
 |    (продолжение)              |                               |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Данные payload (продолжение)              :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Данные payload (продолжение)              |
 +---------------------------------------------------------------+
```

#### Описание полей фрейма

| Поле | Размер | Описание |
|------|--------|----------|
| FIN | 1 бит | Флаг финального фрагмента (1 = последний фрейм) |
| RSV1-3 | 3 бита | Зарезервированы для расширений |
| Opcode | 4 бита | Тип фрейма (текст, бинарные данные, ping, pong, close) |
| MASK | 1 бит | Флаг маскирования (клиент → сервер всегда маскируется) |
| Payload length | 7 бит | Длина данных (или 126/127 для расширенной длины) |
| Masking-key | 32 бита | Ключ маскирования (если MASK = 1) |
| Payload data | переменно | Собственно данные |

#### Типы опкодов

| Опкод | Название | Описание |
|-------|----------|----------|
| 0x0 | Continuation | Продолжение фрагментированного сообщения |
| 0x1 | Text | Текстовые данные (UTF-8) |
| 0x2 | Binary | Бинарные данные |
| 0x8 | Close | Закрытие соединения |
| 0x9 | Ping | Проверка соединения (сервер → клиент) |
| 0xA | Pong | Ответ на ping |

### Жизненный цикл соединения

```
┌─────────────────────────────────────────────────────────────────┐
│                    ЖИЗНЕННЫЙ ЦИКЛ WebSocket                     │
└─────────────────────────────────────────────────────────────────┘

    ┌──────────┐     HTTP Upgrade      ┌──────────┐
    │ ЗАКРЫТО  │ ─────────────────────>│ ОТКРЫТИЕ │
    └──────────┘                       └────┬─────┘
         ^                                  │
         │                                  │ 101 Switching
         │                                  │ Protocols
         │                                  v
         │                            ┌──────────┐
         │                            │ ОТКРЫТО  │<─────────┐
         │                            └────┬─────┘          │
         │                                 │                │
         │     Close Frame                 │ Обмен          │
         └─────────────────────────────────┤ сообщениями    │
                                           │                │
                                           └────────────────┘
```

---

## Примеры кода

### JavaScript клиент (браузер)

```javascript
// Создание WebSocket соединения
const socket = new WebSocket('ws://localhost:8765');

// Обработчик успешного подключения
socket.onopen = function(event) {
    console.log('Соединение установлено');

    // Отправка сообщения на сервер
    socket.send(JSON.stringify({
        type: 'greeting',
        message: 'Привет, сервер!'
    }));
};

// Обработчик входящих сообщений
socket.onmessage = function(event) {
    console.log('Получено сообщение:', event.data);

    try {
        const data = JSON.parse(event.data);
        handleMessage(data);
    } catch (e) {
        console.log('Текстовое сообщение:', event.data);
    }
};

// Обработчик ошибок
socket.onerror = function(error) {
    console.error('Ошибка WebSocket:', error);
};

// Обработчик закрытия соединения
socket.onclose = function(event) {
    if (event.wasClean) {
        console.log(`Соединение закрыто чисто, код: ${event.code}, причина: ${event.reason}`);
    } else {
        console.log('Соединение прервано');
    }
};

// Функция обработки сообщений
function handleMessage(data) {
    switch(data.type) {
        case 'chat':
            displayChatMessage(data);
            break;
        case 'notification':
            showNotification(data);
            break;
        case 'update':
            updateUI(data);
            break;
        default:
            console.log('Неизвестный тип сообщения:', data.type);
    }
}

// Отправка сообщения с проверкой состояния соединения
function sendMessage(message) {
    if (socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify(message));
    } else {
        console.error('Соединение не открыто. Состояние:', socket.readyState);
    }
}

// Константы состояния WebSocket
// WebSocket.CONNECTING = 0 - Соединение устанавливается
// WebSocket.OPEN = 1       - Соединение открыто
// WebSocket.CLOSING = 2    - Соединение закрывается
// WebSocket.CLOSED = 3     - Соединение закрыто
```

### JavaScript клиент с переподключением

```javascript
class WebSocketClient {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            reconnectInterval: 1000,
            maxReconnectInterval: 30000,
            reconnectDecay: 1.5,
            maxReconnectAttempts: null,
            ...options
        };

        this.reconnectAttempts = 0;
        this.handlers = {
            open: [],
            message: [],
            close: [],
            error: []
        };

        this.connect();
    }

    connect() {
        this.socket = new WebSocket(this.url);

        this.socket.onopen = (event) => {
            console.log('WebSocket соединение установлено');
            this.reconnectAttempts = 0;
            this.handlers.open.forEach(handler => handler(event));
        };

        this.socket.onmessage = (event) => {
            this.handlers.message.forEach(handler => handler(event));
        };

        this.socket.onclose = (event) => {
            this.handlers.close.forEach(handler => handler(event));
            this.reconnect();
        };

        this.socket.onerror = (error) => {
            this.handlers.error.forEach(handler => handler(error));
        };
    }

    reconnect() {
        if (this.options.maxReconnectAttempts !== null &&
            this.reconnectAttempts >= this.options.maxReconnectAttempts) {
            console.log('Достигнуто максимальное количество попыток переподключения');
            return;
        }

        const timeout = Math.min(
            this.options.reconnectInterval * Math.pow(
                this.options.reconnectDecay,
                this.reconnectAttempts
            ),
            this.options.maxReconnectInterval
        );

        console.log(`Переподключение через ${timeout}мс...`);

        setTimeout(() => {
            this.reconnectAttempts++;
            this.connect();
        }, timeout);
    }

    on(event, handler) {
        if (this.handlers[event]) {
            this.handlers[event].push(handler);
        }
        return this;
    }

    send(data) {
        if (this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(typeof data === 'string' ? data : JSON.stringify(data));
            return true;
        }
        return false;
    }

    close() {
        this.options.maxReconnectAttempts = 0;
        this.socket.close();
    }
}

// Использование
const client = new WebSocketClient('ws://localhost:8765')
    .on('open', () => console.log('Подключено!'))
    .on('message', (event) => console.log('Сообщение:', event.data))
    .on('close', () => console.log('Отключено'))
    .on('error', (error) => console.error('Ошибка:', error));

client.send({ type: 'hello', content: 'Привет!' });
```

### Python сервер (asyncio + websockets)

```python
import asyncio
import websockets
import json
from datetime import datetime
from typing import Set

# Множество подключённых клиентов
connected_clients: Set[websockets.WebSocketServerProtocol] = set()


async def register(websocket: websockets.WebSocketServerProtocol) -> None:
    """Регистрация нового клиента."""
    connected_clients.add(websocket)
    print(f"Клиент подключён. Всего клиентов: {len(connected_clients)}")


async def unregister(websocket: websockets.WebSocketServerProtocol) -> None:
    """Удаление отключившегося клиента."""
    connected_clients.discard(websocket)
    print(f"Клиент отключён. Всего клиентов: {len(connected_clients)}")


async def broadcast(message: str) -> None:
    """Отправка сообщения всем подключённым клиентам."""
    if connected_clients:
        await asyncio.gather(
            *[client.send(message) for client in connected_clients],
            return_exceptions=True
        )


async def handler(
    websocket: websockets.WebSocketServerProtocol,
    path: str
) -> None:
    """Обработчик WebSocket соединений."""
    await register(websocket)

    try:
        # Отправляем приветственное сообщение
        welcome = json.dumps({
            "type": "welcome",
            "message": "Добро пожаловать на сервер!",
            "timestamp": datetime.now().isoformat()
        })
        await websocket.send(welcome)

        # Обрабатываем входящие сообщения
        async for message in websocket:
            try:
                data = json.loads(message)
                print(f"Получено: {data}")

                # Обработка разных типов сообщений
                if data.get("type") == "chat":
                    # Рассылаем сообщение чата всем клиентам
                    broadcast_message = json.dumps({
                        "type": "chat",
                        "user": data.get("user", "Аноним"),
                        "message": data.get("message", ""),
                        "timestamp": datetime.now().isoformat()
                    })
                    await broadcast(broadcast_message)

                elif data.get("type") == "ping":
                    # Отвечаем на ping
                    pong = json.dumps({
                        "type": "pong",
                        "timestamp": datetime.now().isoformat()
                    })
                    await websocket.send(pong)

                else:
                    # Эхо для неизвестных типов
                    response = json.dumps({
                        "type": "echo",
                        "original": data,
                        "timestamp": datetime.now().isoformat()
                    })
                    await websocket.send(response)

            except json.JSONDecodeError:
                # Обработка не-JSON сообщений
                response = json.dumps({
                    "type": "echo",
                    "message": message,
                    "timestamp": datetime.now().isoformat()
                })
                await websocket.send(response)

    except websockets.exceptions.ConnectionClosedError:
        print("Соединение закрыто с ошибкой")
    except websockets.exceptions.ConnectionClosedOK:
        print("Соединение закрыто нормально")
    finally:
        await unregister(websocket)


async def main():
    """Запуск WebSocket сервера."""
    server = await websockets.serve(
        handler,
        "localhost",
        8765,
        ping_interval=20,      # Отправлять ping каждые 20 секунд
        ping_timeout=20,       # Таймаут ожидания pong
        close_timeout=10       # Таймаут закрытия соединения
    )

    print("WebSocket сервер запущен на ws://localhost:8765")

    await server.wait_closed()


if __name__ == "__main__":
    asyncio.run(main())
```

### Python сервер с комнатами (чат)

```python
import asyncio
import websockets
import json
from datetime import datetime
from typing import Dict, Set
from dataclasses import dataclass, field


@dataclass
class Room:
    """Класс комнаты чата."""
    name: str
    clients: Set[websockets.WebSocketServerProtocol] = field(default_factory=set)

    async def broadcast(self, message: str, exclude=None):
        """Отправка сообщения всем клиентам комнаты."""
        for client in self.clients:
            if client != exclude:
                try:
                    await client.send(message)
                except websockets.exceptions.ConnectionClosed:
                    pass


class ChatServer:
    """Сервер чата с поддержкой комнат."""

    def __init__(self):
        self.rooms: Dict[str, Room] = {}
        self.client_rooms: Dict[websockets.WebSocketServerProtocol, str] = {}

    def get_or_create_room(self, room_name: str) -> Room:
        """Получить или создать комнату."""
        if room_name not in self.rooms:
            self.rooms[room_name] = Room(name=room_name)
            print(f"Создана комната: {room_name}")
        return self.rooms[room_name]

    async def join_room(
        self,
        websocket: websockets.WebSocketServerProtocol,
        room_name: str,
        username: str
    ):
        """Присоединение клиента к комнате."""
        # Покидаем текущую комнату, если есть
        await self.leave_current_room(websocket, username)

        room = self.get_or_create_room(room_name)
        room.clients.add(websocket)
        self.client_rooms[websocket] = room_name

        # Уведомляем комнату о новом участнике
        notification = json.dumps({
            "type": "system",
            "message": f"{username} присоединился к комнате",
            "room": room_name,
            "timestamp": datetime.now().isoformat()
        })
        await room.broadcast(notification)

        print(f"{username} присоединился к комнате {room_name}")

    async def leave_current_room(
        self,
        websocket: websockets.WebSocketServerProtocol,
        username: str
    ):
        """Покинуть текущую комнату."""
        if websocket in self.client_rooms:
            room_name = self.client_rooms[websocket]
            room = self.rooms.get(room_name)

            if room:
                room.clients.discard(websocket)

                # Уведомляем остальных участников
                notification = json.dumps({
                    "type": "system",
                    "message": f"{username} покинул комнату",
                    "room": room_name,
                    "timestamp": datetime.now().isoformat()
                })
                await room.broadcast(notification)

                # Удаляем пустую комнату
                if not room.clients:
                    del self.rooms[room_name]
                    print(f"Комната {room_name} удалена (пустая)")

            del self.client_rooms[websocket]

    async def send_message(
        self,
        websocket: websockets.WebSocketServerProtocol,
        username: str,
        message: str
    ):
        """Отправка сообщения в текущую комнату."""
        if websocket not in self.client_rooms:
            error = json.dumps({
                "type": "error",
                "message": "Вы не находитесь в комнате"
            })
            await websocket.send(error)
            return

        room_name = self.client_rooms[websocket]
        room = self.rooms[room_name]

        chat_message = json.dumps({
            "type": "chat",
            "user": username,
            "message": message,
            "room": room_name,
            "timestamp": datetime.now().isoformat()
        })
        await room.broadcast(chat_message)

    async def handler(
        self,
        websocket: websockets.WebSocketServerProtocol,
        path: str
    ):
        """Обработчик соединений."""
        username = None

        try:
            async for message in websocket:
                try:
                    data = json.loads(message)
                    action = data.get("action")

                    if action == "set_username":
                        username = data.get("username", "Аноним")
                        response = json.dumps({
                            "type": "system",
                            "message": f"Имя установлено: {username}"
                        })
                        await websocket.send(response)

                    elif action == "join":
                        if not username:
                            username = "Аноним"
                        room_name = data.get("room", "general")
                        await self.join_room(websocket, room_name, username)

                    elif action == "message":
                        if not username:
                            username = "Аноним"
                        await self.send_message(
                            websocket,
                            username,
                            data.get("text", "")
                        )

                    elif action == "leave":
                        if not username:
                            username = "Аноним"
                        await self.leave_current_room(websocket, username)

                except json.JSONDecodeError:
                    error = json.dumps({
                        "type": "error",
                        "message": "Неверный формат JSON"
                    })
                    await websocket.send(error)

        except websockets.exceptions.ConnectionClosed:
            pass
        finally:
            if username:
                await self.leave_current_room(websocket, username)


async def main():
    server = ChatServer()

    async with websockets.serve(server.handler, "localhost", 8765):
        print("Чат-сервер запущен на ws://localhost:8765")
        await asyncio.Future()  # Работаем вечно


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Преимущества и недостатки WebSocket

| Преимущества | Недостатки |
|--------------|------------|
| **Двунаправленная связь** — сервер может инициировать отправку данных | **Сложность масштабирования** — требуется сохранение состояния соединений |
| **Низкая задержка** — нет накладных расходов на установку соединения | **Проблемы с прокси** — некоторые прокси-серверы не поддерживают WebSocket |
| **Минимальные накладные расходы** — заголовки фреймов очень компактны | **Нет встроенного механизма восстановления** — нужно реализовывать переподключение |
| **Постоянное соединение** — не нужно переустанавливать соединение | **Сложность отладки** — инструменты разработчика менее удобны, чем для HTTP |
| **Поддержка текста и бинарных данных** — универсальность передачи | **Потребление ресурсов** — каждое соединение занимает память и файловый дескриптор |
| **Широкая поддержка** — работает во всех современных браузерах | **Firewall и NAT** — могут блокировать или прерывать долгоживущие соединения |
| **Нативная поддержка** — не требует дополнительных библиотек в браузере | **Безопасность** — требует дополнительных мер защиты (валидация, авторизация) |

### Подробнее о преимуществах

#### 1. Минимальные накладные расходы

Сравнение размера заголовков:

```
HTTP запрос (примерно):
┌─────────────────────────────────────────────────────────────┐
│ GET /api/messages HTTP/1.1                                  │
│ Host: example.com                                           │
│ User-Agent: Mozilla/5.0 (...)                               │
│ Accept: application/json                                    │
│ Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8                 │
│ Accept-Encoding: gzip, deflate, br                          │
│ Cookie: session_id=abc123; user_token=xyz789; ...           │
│ Connection: keep-alive                                      │
│ ... и другие заголовки                                      │
└─────────────────────────────────────────────────────────────┘
Размер: 500-2000+ байт на КАЖДЫЙ запрос

WebSocket фрейм:
┌───────────────────────────┐
│ 2-14 байт заголовка       │
│ + данные                  │
└───────────────────────────┘
Размер заголовка: 2-14 байт
```

#### 2. Низкая задержка

```
HTTP Polling:
Клиент ─────> Сервер (запрос)     ~50ms RTT
       <───── (ответ)             ~50ms RTT
       [пауза 1-5 секунд]
       ─────> (новый запрос)      ~50ms RTT
       <───── (ответ)             ~50ms RTT

WebSocket:
Клиент <═══> Сервер               ~1-5ms на сообщение
(постоянное соединение)           Нет накладных расходов
```

---

## Когда использовать WebSocket

### Идеальные сценарии использования

#### 1. Чаты и мессенджеры

```javascript
// Пример структуры сообщений чата
const chatMessage = {
    type: 'chat',
    room: 'general',
    user: 'Иван',
    message: 'Привет всем!',
    timestamp: '2024-01-15T10:30:00Z'
};

// Типы событий
const eventTypes = {
    MESSAGE: 'message',          // Новое сообщение
    TYPING: 'typing',            // Пользователь печатает
    PRESENCE: 'presence',        // Онлайн/оффлайн статус
    READ: 'read',                // Сообщение прочитано
    REACTION: 'reaction'         // Реакция на сообщение
};
```

#### 2. Онлайн-игры

```python
# Пример игрового состояния
game_state = {
    "type": "game_update",
    "players": [
        {"id": 1, "x": 100, "y": 200, "health": 100},
        {"id": 2, "x": 300, "y": 150, "health": 75}
    ],
    "objects": [
        {"id": 101, "type": "bullet", "x": 150, "y": 180, "dx": 5, "dy": 0}
    ],
    "tick": 12345,
    "timestamp": 1705312200.123
}

# Типичная частота обновлений
# - Шутеры: 60-128 тиков/сек
# - Стратегии: 10-30 тиков/сек
# - Пошаговые игры: по событиям
```

#### 3. Финансовые приложения (биржевые котировки)

```javascript
// Поток котировок
const priceUpdate = {
    type: 'price',
    symbol: 'AAPL',
    bid: 182.50,
    ask: 182.52,
    last: 182.51,
    volume: 1250000,
    timestamp: 1705312200123
};

// Книга заявок (order book)
const orderBook = {
    type: 'orderbook',
    symbol: 'BTC/USD',
    bids: [
        [42150.00, 1.5],
        [42149.50, 2.3],
        [42149.00, 0.8]
    ],
    asks: [
        [42151.00, 1.2],
        [42151.50, 3.1],
        [42152.00, 2.0]
    ],
    timestamp: 1705312200123
};
```

#### 4. Совместное редактирование

```javascript
// Операционная трансформация (OT)
const operation = {
    type: 'operation',
    documentId: 'doc-123',
    userId: 'user-456',
    operations: [
        { type: 'retain', count: 10 },
        { type: 'insert', text: 'Привет' },
        { type: 'retain', count: 5 },
        { type: 'delete', count: 3 }
    ],
    version: 42
};

// Позиции курсоров других пользователей
const cursorUpdate = {
    type: 'cursor',
    documentId: 'doc-123',
    userId: 'user-789',
    userName: 'Мария',
    position: 156,
    selection: { start: 156, end: 180 }
};
```

#### 5. IoT и мониторинг

```python
# Данные с датчиков
sensor_data = {
    "type": "sensor_reading",
    "device_id": "temp-sensor-001",
    "readings": {
        "temperature": 23.5,
        "humidity": 45.2,
        "pressure": 1013.25
    },
    "battery": 87,
    "timestamp": "2024-01-15T10:30:00Z"
}

# Алерты
alert = {
    "type": "alert",
    "severity": "warning",
    "device_id": "temp-sensor-001",
    "message": "Температура превысила порог",
    "value": 28.5,
    "threshold": 27.0,
    "timestamp": "2024-01-15T10:30:00Z"
}
```

### Когда НЕ использовать WebSocket

- **Простые CRUD операции** — используйте REST API
- **Редкие обновления** — достаточно периодических HTTP запросов
- **Только клиент → сервер** — достаточно обычного HTTP
- **Кэшируемые данные** — HTTP кэширование эффективнее
- **SEO-важный контент** — поисковики не индексируют WebSocket данные

---

## Сравнение с HTTP Polling

### Виды polling

```
┌─────────────────────────────────────────────────────────────────────┐
│                    МЕТОДЫ ПОЛУЧЕНИЯ ДАННЫХ                          │
└─────────────────────────────────────────────────────────────────────┘

1. SHORT POLLING (Короткий опрос)
   ┌────────┐                                    ┌────────┐
   │ Клиент │                                    │ Сервер │
   └───┬────┘                                    └───┬────┘
       │ ──── GET /updates ────>                     │
       │ <─── "нет данных" ─────                     │  ~50ms
       │                                             │
       │        [пауза 5 сек]                        │
       │                                             │
       │ ──── GET /updates ────>                     │
       │ <─── "нет данных" ─────                     │  ~50ms
       │                                             │
       │        [пауза 5 сек]                        │
       │                                             │
       │ ──── GET /updates ────>                     │
       │ <─── {данные} ─────────                     │  ~50ms

   Проблема: много пустых запросов, задержка до 5 сек

2. LONG POLLING (Длинный опрос)
   ┌────────┐                                    ┌────────┐
   │ Клиент │                                    │ Сервер │
   └───┬────┘                                    └───┬────┘
       │ ──── GET /updates ────>                     │
       │                [сервер ждёт данные...]      │
       │                    [20-30 секунд]           │
       │ <─── {данные} ─────────                     │
       │                                             │
       │ ──── GET /updates ────>                     │  (сразу)
       │                [сервер ждёт...]             │

   Лучше: меньше запросов, но соединение занято

3. SERVER-SENT EVENTS (SSE)
   ┌────────┐                                    ┌────────┐
   │ Клиент │                                    │ Сервер │
   └───┬────┘                                    └───┬────┘
       │ ──── GET /events ────>                      │
       │ <═══ event: message ════                    │
       │ <═══ event: message ════                    │
       │ <═══ event: update ═════                    │
       │ <═══ event: message ════                    │
       │      (постоянный поток)                     │

   Хорошо: сервер может push'ить, но только текст

4. WEBSOCKET
   ┌────────┐                                    ┌────────┐
   │ Клиент │                                    │ Сервер │
   └───┬────┘                                    └───┬────┘
       │ ═════ Handshake ════════>                   │
       │ <════════════════════════                   │
       │                                             │
       │ <════ сообщение ════════                    │
       │ ════ сообщение ════════>                    │
       │ <════ сообщение ════════                    │
       │ ════ сообщение ════════>                    │
       │     (полный дуплекс)                        │

   Лучше всего: двунаправленный обмен в реальном времени
```

### Сравнительная таблица

| Характеристика | Short Polling | Long Polling | SSE | WebSocket |
|----------------|---------------|--------------|-----|-----------|
| **Направление** | Клиент → Сервер | Клиент → Сервер | Сервер → Клиент | Двунаправленное |
| **Задержка** | Высокая (интервал) | Средняя | Низкая | Очень низкая |
| **Нагрузка на сервер** | Высокая | Средняя | Низкая | Низкая |
| **Нагрузка на сеть** | Высокая | Средняя | Низкая | Очень низкая |
| **Сложность реализации** | Простая | Средняя | Простая | Средняя |
| **Поддержка браузерами** | 100% | 100% | ~97% | ~98% |
| **Бинарные данные** | Да (Base64) | Да (Base64) | Нет | Да (нативно) |
| **Автопереподключение** | Нет | Нет | Да | Нет (нужно реализовать) |
| **HTTP/2 совместимость** | Да | Да | Да | Нет (отдельное соединение) |

### Расчёт нагрузки

Пример: 10 000 активных пользователей, требуется обновление каждую секунду.

```
SHORT POLLING (интервал 1 сек):
─────────────────────────────────
Запросов в секунду: 10,000
Размер запроса: ~500 байт
Размер ответа: ~200 байт
Трафик: 10,000 × 700 = 7 МБ/сек
Соединений TCP/сек: 10,000

LONG POLLING:
─────────────────────────────────
Соединений одновременно: 10,000
Размер ответа: ~200 байт
Трафик при событии: ~2 МБ (каждому)
Сложность: нужно управлять ожидающими соединениями

WEBSOCKET:
─────────────────────────────────
Соединений: 10,000 (постоянных)
Размер сообщения: ~50 байт (меньше заголовков)
Трафик при событии: ~500 КБ (каждому)
Нет накладных расходов на установку соединения
```

---

## Безопасность WebSocket

### Защита соединения

```python
import asyncio
import websockets
import ssl
import jwt
from functools import wraps

# Создание SSL контекста для WSS
ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ssl_context.load_cert_chain('certificate.pem', 'private_key.pem')


async def authenticate(websocket, path):
    """Аутентификация при подключении."""
    try:
        # Ожидаем токен в первом сообщении
        auth_message = await asyncio.wait_for(
            websocket.recv(),
            timeout=10.0
        )

        data = json.loads(auth_message)
        token = data.get('token')

        if not token:
            await websocket.close(4001, 'Токен не предоставлен')
            return None

        # Проверяем JWT токен
        try:
            payload = jwt.decode(
                token,
                'secret_key',
                algorithms=['HS256']
            )
            return payload['user_id']
        except jwt.InvalidTokenError:
            await websocket.close(4002, 'Неверный токен')
            return None

    except asyncio.TimeoutError:
        await websocket.close(4003, 'Таймаут аутентификации')
        return None


async def secure_handler(websocket, path):
    """Защищённый обработчик."""
    user_id = await authenticate(websocket, path)

    if not user_id:
        return

    print(f"Пользователь {user_id} авторизован")

    # Дальнейшая обработка...
    async for message in websocket:
        # Валидация входящих данных
        try:
            data = json.loads(message)
            # Проверка и санитизация данных
            sanitized = sanitize_input(data)
            await process_message(websocket, user_id, sanitized)
        except (json.JSONDecodeError, ValueError) as e:
            await websocket.send(json.dumps({
                'error': 'Неверный формат данных'
            }))


def sanitize_input(data):
    """Санитизация входных данных."""
    if not isinstance(data, dict):
        raise ValueError("Ожидается объект")

    # Ограничение длины сообщения
    if 'message' in data:
        if len(data['message']) > 1000:
            data['message'] = data['message'][:1000]
        # Удаление опасных символов
        data['message'] = data['message'].replace('<', '&lt;').replace('>', '&gt;')

    return data


# Запуск с SSL
async def main():
    server = await websockets.serve(
        secure_handler,
        "0.0.0.0",
        8765,
        ssl=ssl_context
    )
    await server.wait_closed()
```

### Рекомендации по безопасности

1. **Всегда используйте WSS (WebSocket Secure)** в продакшене
2. **Аутентифицируйте** пользователей при подключении
3. **Валидируйте** все входящие данные
4. **Ограничивайте** размер сообщений
5. **Реализуйте rate limiting** для предотвращения DDoS
6. **Проверяйте Origin** заголовок для защиты от CSRF
7. **Используйте таймауты** для неактивных соединений

---

## Масштабирование WebSocket

### Горизонтальное масштабирование с Redis Pub/Sub

```
┌─────────────────────────────────────────────────────────────┐
│                    АРХИТЕКТУРА                              │
└─────────────────────────────────────────────────────────────┘

                        ┌──────────────┐
                        │  Балансировщик│
                        │  нагрузки    │
                        └──────┬───────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼─────┐        ┌─────▼─────┐        ┌─────▼─────┐
    │ WS Server │        │ WS Server │        │ WS Server │
    │     1     │        │     2     │        │     3     │
    └─────┬─────┘        └─────┬─────┘        └─────┬─────┘
          │                    │                    │
          └────────────────────┼────────────────────┘
                               │
                        ┌──────▼───────┐
                        │    Redis     │
                        │   Pub/Sub    │
                        └──────────────┘
```

```python
import asyncio
import websockets
import redis.asyncio as redis
import json


class ScalableWebSocketServer:
    """WebSocket сервер с поддержкой масштабирования через Redis."""

    def __init__(self, redis_url='redis://localhost'):
        self.redis_url = redis_url
        self.redis = None
        self.pubsub = None
        self.clients = set()

    async def connect_redis(self):
        """Подключение к Redis."""
        self.redis = await redis.from_url(self.redis_url)
        self.pubsub = self.redis.pubsub()
        await self.pubsub.subscribe('broadcast')

    async def redis_listener(self):
        """Слушатель Redis Pub/Sub."""
        async for message in self.pubsub.listen():
            if message['type'] == 'message':
                data = message['data'].decode('utf-8')
                await self.broadcast_local(data)

    async def broadcast_local(self, message):
        """Отправка сообщения локальным клиентам."""
        if self.clients:
            await asyncio.gather(
                *[client.send(message) for client in self.clients],
                return_exceptions=True
            )

    async def publish(self, message):
        """Публикация сообщения через Redis."""
        await self.redis.publish('broadcast', message)

    async def handler(self, websocket, path):
        """Обработчик соединений."""
        self.clients.add(websocket)
        try:
            async for message in websocket:
                # Публикуем через Redis для всех серверов
                await self.publish(message)
        finally:
            self.clients.discard(websocket)

    async def run(self, host='localhost', port=8765):
        """Запуск сервера."""
        await self.connect_redis()

        # Запускаем слушатель Redis
        asyncio.create_task(self.redis_listener())

        # Запускаем WebSocket сервер
        server = await websockets.serve(self.handler, host, port)
        print(f"Сервер запущен на ws://{host}:{port}")

        await server.wait_closed()


if __name__ == "__main__":
    server = ScalableWebSocketServer()
    asyncio.run(server.run())
```

---

## Полезные библиотеки

### Python

| Библиотека | Описание | Использование |
|------------|----------|---------------|
| `websockets` | Асинхронная реализация WebSocket | Низкоуровневый контроль |
| `aiohttp` | HTTP клиент/сервер с поддержкой WS | Веб-приложения |
| `FastAPI` | Веб-фреймворк с WebSocket | API + WebSocket |
| `channels` | Django Channels для async | Django проекты |
| `socket.io` | python-socketio | Socket.IO совместимость |

### JavaScript/TypeScript

| Библиотека | Описание | Использование |
|------------|----------|---------------|
| `ws` | Node.js WebSocket библиотека | Серверная разработка |
| `socket.io` | Популярная библиотека с fallback | Универсальные приложения |
| `sockjs` | Эмуляция WebSocket | Поддержка старых браузеров |
| `reconnecting-websocket` | Автопереподключение | Клиентские приложения |

---

## Отладка WebSocket

### Chrome DevTools

1. Откройте DevTools (F12)
2. Перейдите на вкладку "Network"
3. Отфильтруйте по "WS"
4. Выберите соединение для просмотра сообщений

### Консольные инструменты

```bash
# websocat - "curl для WebSocket"
websocat ws://localhost:8765

# wscat - Node.js клиент
wscat -c ws://localhost:8765

# Отправка сообщения
> {"type": "hello", "message": "test"}
```

---

## Заключение

WebSocket — мощный протокол для создания приложений реального времени. Он обеспечивает эффективную двунаправленную связь между клиентом и сервером с минимальными накладными расходами.

### Ключевые моменты

1. **Используйте WebSocket**, когда нужна низкая задержка и двунаправленный обмен
2. **Не используйте WebSocket** для простых CRUD операций
3. **Всегда реализуйте** механизм переподключения
4. **Продумывайте масштабирование** заранее (Redis Pub/Sub)
5. **Не забывайте о безопасности** — WSS, аутентификация, валидация

### Дополнительные ресурсы

- [RFC 6455 — The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [MDN Web Docs — WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [websockets (Python)](https://websockets.readthedocs.io/)
- [Socket.IO](https://socket.io/)

---

[prev: 03-next-steps](../18-message-brokers/02-kafka/12-stream-processing/03-next-steps.md) | [next: 02-server-sent-events](./02-server-sent-events.md)
