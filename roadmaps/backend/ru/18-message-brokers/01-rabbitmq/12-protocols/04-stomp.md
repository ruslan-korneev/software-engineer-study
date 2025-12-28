# STOMP

[prev: 03-mqtt](./03-mqtt.md) | [next: 01-what-is-kafka](../../02-kafka/01-introduction/01-what-is-kafka.md)

---

## Введение

**STOMP** (Simple Text Oriented Messaging Protocol) — это простой текстовый протокол обмена сообщениями. В отличие от бинарных протоколов (AMQP, MQTT), STOMP использует человекочитаемый формат, что делает его удобным для отладки и интеграции с веб-приложениями. RabbitMQ поддерживает STOMP через плагин `rabbitmq_stomp`.

## Характеристики STOMP

### Особенности протокола

| Характеристика | Описание |
|----------------|----------|
| **Текстовый формат** | Команды и данные в читаемом виде |
| **Простота** | Минимальный набор команд |
| **Web-friendly** | Хорошо работает через WebSocket |
| **Межплатформенность** | Легко реализовать на любом языке |
| **Отладка** | Можно отлаживать обычным telnet |

### Сравнение с другими протоколами

| Аспект | STOMP | AMQP 0-9-1 | MQTT |
|--------|-------|------------|------|
| Формат | Текстовый | Бинарный | Бинарный |
| Сложность | Простой | Сложный | Простой |
| Overhead | Больше | Меньше | Минимальный |
| WebSocket | Отлично | Требует адаптации | Хорошо |
| Отладка | Легко | Требует инструменты | Требует инструменты |

## Структура протокола

### Формат фрейма

Каждый STOMP фрейм имеет следующую структуру:

```
COMMAND
header1:value1
header2:value2

Body^@
```

Где `^@` — это NULL-символ (ASCII 0), обозначающий конец фрейма.

### Пример фрейма

```
SEND
destination:/queue/orders
content-type:application/json
content-length:27

{"order_id": 123, "qty": 5}^@
```

## Команды STOMP

### Клиентские команды

| Команда | Описание |
|---------|----------|
| `CONNECT` / `STOMP` | Установка соединения |
| `SEND` | Отправка сообщения |
| `SUBSCRIBE` | Подписка на destination |
| `UNSUBSCRIBE` | Отписка от destination |
| `ACK` | Подтверждение сообщения |
| `NACK` | Отклонение сообщения |
| `BEGIN` | Начало транзакции |
| `COMMIT` | Фиксация транзакции |
| `ABORT` | Откат транзакции |
| `DISCONNECT` | Закрытие соединения |

### Серверные команды

| Команда | Описание |
|---------|----------|
| `CONNECTED` | Подтверждение подключения |
| `MESSAGE` | Доставка сообщения |
| `RECEIPT` | Подтверждение обработки команды |
| `ERROR` | Сообщение об ошибке |

## Процесс подключения

```
Client                              Server
  |                                    |
  |-------- CONNECT ------------------>|
  |          login:guest               |
  |          passcode:guest            |
  |          accept-version:1.2        |
  |                                    |
  |<------- CONNECTED -----------------|
  |          version:1.2               |
  |          server:RabbitMQ           |
  |                                    |
  |-------- SUBSCRIBE ---------------->|
  |          destination:/queue/test   |
  |          id:sub-0                  |
  |          ack:client                |
  |                                    |
  |<------- MESSAGE -------------------|
  |          subscription:sub-0        |
  |          message-id:msg-123        |
  |          destination:/queue/test   |
  |          Hello, STOMP!             |
  |                                    |
  |-------- ACK ---------------------->|
  |          id:msg-123                |
```

## Установка плагина в RabbitMQ

### Включение плагина

```bash
# Включить STOMP плагин
rabbitmq-plugins enable rabbitmq_stomp

# Опционально: включить WebSocket для STOMP
rabbitmq-plugins enable rabbitmq_web_stomp

# Проверить статус
rabbitmq-plugins list | grep stomp

# Перезапустить RabbitMQ
sudo systemctl restart rabbitmq-server
```

### Конфигурация

```bash
# rabbitmq.conf

# Порт для STOMP (по умолчанию 61613)
stomp.listeners.tcp.default = 61613

# Порт для STOMP over TLS
stomp.listeners.ssl.default = 61614

# WebSocket порт (требует rabbitmq_web_stomp)
web_stomp.tcp.port = 15674

# Виртуальный хост по умолчанию
stomp.default_vhost = /

# Пользователь по умолчанию
stomp.default_user = guest
stomp.default_pass = guest

# Разрешить гостевой доступ
stomp.allow_anonymous = false

# Heartbeat (в миллисекундах)
stomp.heartbeat_send = 10000
stomp.heartbeat_receive = 10000

# Максимальный размер фрейма
stomp.max_frame_size = 1048576
```

### TLS конфигурация

```bash
# rabbitmq.conf

stomp.listeners.ssl.default = 61614
ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile = /path/to/server_certificate.pem
ssl_options.keyfile = /path/to/server_key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

## Destinations (Назначения)

### Маппинг на RabbitMQ

STOMP destinations маппятся на сущности RabbitMQ:

| STOMP Destination | RabbitMQ Entity |
|-------------------|-----------------|
| `/queue/name` | Queue с именем "name" |
| `/topic/name` | Topic exchange "amq.topic" с routing key "name" |
| `/exchange/name` | Exchange с пустым routing key |
| `/exchange/name/routing.key` | Exchange с указанным routing key |
| `/amq/queue/name` | Queue напрямую (без автосоздания) |
| `/temp-queue/name` | Временная очередь |

### Примеры destinations

```python
# Отправка в очередь (автосоздание)
destination = "/queue/orders"

# Отправка в topic exchange
destination = "/topic/events.user.created"

# Отправка в именованный exchange
destination = "/exchange/my_exchange/my.routing.key"

# Временная очередь для reply-to
destination = "/temp-queue/reply-to-123"
```

## Примеры кода

### Python (stomp.py)

```python
import stomp
import json
import time

# Listener для обработки сообщений
class MyListener(stomp.ConnectionListener):
    def __init__(self, conn):
        self.conn = conn

    def on_error(self, frame):
        print(f'Ошибка: {frame.body}')

    def on_message(self, frame):
        print(f'Получено сообщение:')
        print(f'  Destination: {frame.headers["destination"]}')
        print(f'  Message ID: {frame.headers["message-id"]}')
        print(f'  Body: {frame.body}')

        # Подтверждение (если ack=client)
        self.conn.ack(frame.headers['message-id'],
                      frame.headers['subscription'])

    def on_connected(self, frame):
        print(f'Подключено к серверу: {frame.headers.get("server", "unknown")}')

    def on_disconnected(self):
        print('Отключено')

# Создание подключения
conn = stomp.Connection([('localhost', 61613)])
conn.set_listener('', MyListener(conn))

# Подключение
conn.connect(
    username='guest',
    passcode='guest',
    wait=True,
    headers={
        'accept-version': '1.2',
        'heart-beat': '10000,10000'
    }
)

# Подписка на очередь
conn.subscribe(
    destination='/queue/orders',
    id='sub-1',
    ack='client',  # 'auto', 'client', 'client-individual'
    headers={
        'prefetch-count': '10'
    }
)

# Отправка сообщения
conn.send(
    destination='/queue/orders',
    body=json.dumps({
        'order_id': 123,
        'product': 'Widget',
        'quantity': 5
    }),
    headers={
        'content-type': 'application/json',
        'custom-header': 'custom-value'
    }
)

# Отправка с подтверждением (receipt)
conn.send(
    destination='/queue/orders',
    body='Important message',
    headers={
        'receipt': 'receipt-1'
    }
)

# Ожидание сообщений
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    conn.disconnect()
```

### Python (с транзакциями)

```python
import stomp
import json

conn = stomp.Connection([('localhost', 61613)])
conn.connect('guest', 'guest', wait=True)

# Начало транзакции
tx_id = 'tx-123'
conn.begin(transaction=tx_id)

try:
    # Отправка сообщений в транзакции
    conn.send(
        destination='/queue/orders',
        body=json.dumps({'action': 'create', 'id': 1}),
        headers={'transaction': tx_id}
    )

    conn.send(
        destination='/queue/orders',
        body=json.dumps({'action': 'update', 'id': 1}),
        headers={'transaction': tx_id}
    )

    # Фиксация транзакции
    conn.commit(transaction=tx_id)
    print("Транзакция зафиксирована")

except Exception as e:
    # Откат транзакции при ошибке
    conn.abort(transaction=tx_id)
    print(f"Транзакция отменена: {e}")

finally:
    conn.disconnect()
```

### Node.js (@stomp/stompjs)

```javascript
const { Client } = require('@stomp/stompjs');
const WebSocket = require('ws');

// Клиент для WebSocket
Object.assign(global, { WebSocket });

const client = new Client({
    brokerURL: 'ws://localhost:15674/ws',
    connectHeaders: {
        login: 'guest',
        passcode: 'guest'
    },
    heartbeatIncoming: 10000,
    heartbeatOutgoing: 10000,
    reconnectDelay: 5000
});

client.onConnect = function (frame) {
    console.log('Подключено:', frame.headers['server']);

    // Подписка
    const subscription = client.subscribe('/queue/orders', function (message) {
        console.log('Получено:', message.body);

        // Подтверждение
        message.ack();
    }, {
        ack: 'client'
    });

    // Отправка сообщения
    client.publish({
        destination: '/queue/orders',
        body: JSON.stringify({ order_id: 123 }),
        headers: {
            'content-type': 'application/json'
        }
    });
};

client.onStompError = function (frame) {
    console.error('Ошибка брокера:', frame.headers['message']);
    console.error('Детали:', frame.body);
};

client.onWebSocketClose = function () {
    console.log('WebSocket закрыт');
};

// Активация клиента
client.activate();

// Для отключения
// client.deactivate();
```

### JavaScript (Browser)

```html
<!DOCTYPE html>
<html>
<head>
    <title>STOMP WebSocket Client</title>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>STOMP Dashboard</h1>
    <div>
        <input type="text" id="message" placeholder="Сообщение">
        <button onclick="sendMessage()">Отправить</button>
    </div>
    <div id="messages"></div>

    <script>
        const client = new StompJs.Client({
            brokerURL: 'ws://localhost:15674/ws',
            connectHeaders: {
                login: 'guest',
                passcode: 'guest'
            },
            heartbeatIncoming: 10000,
            heartbeatOutgoing: 10000
        });

        client.onConnect = function (frame) {
            console.log('Подключено');
            document.getElementById('messages').innerHTML += '<p>Подключено к серверу</p>';

            // Подписка
            client.subscribe('/topic/chat', function (message) {
                const div = document.getElementById('messages');
                const p = document.createElement('p');
                p.textContent = message.body;
                div.appendChild(p);
            });
        };

        client.onStompError = function (frame) {
            console.error('Ошибка:', frame.body);
        };

        client.activate();

        function sendMessage() {
            const input = document.getElementById('message');
            client.publish({
                destination: '/topic/chat',
                body: input.value
            });
            input.value = '';
        }
    </script>
</body>
</html>
```

### Go (go-stomp/stomp)

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/go-stomp/stomp/v3"
)

type Order struct {
    OrderID int    `json:"order_id"`
    Product string `json:"product"`
    Qty     int    `json:"qty"`
}

func main() {
    // Подключение
    conn, err := stomp.Dial("tcp", "localhost:61613",
        stomp.ConnOpt.Login("guest", "guest"),
        stomp.ConnOpt.HeartBeat(10*time.Second, 10*time.Second),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Disconnect()

    fmt.Println("Подключено к STOMP брокеру")

    // Подписка
    sub, err := conn.Subscribe("/queue/orders", stomp.AckClient)
    if err != nil {
        log.Fatal(err)
    }

    // Обработка сообщений в горутине
    go func() {
        for msg := range sub.C {
            if msg.Err != nil {
                log.Printf("Ошибка: %v", msg.Err)
                continue
            }

            fmt.Printf("Получено: %s\n", string(msg.Body))
            fmt.Printf("  Destination: %s\n", msg.Destination)

            // Подтверждение
            err := conn.Ack(msg)
            if err != nil {
                log.Printf("Ошибка ACK: %v", err)
            }
        }
    }()

    // Отправка сообщений
    for i := 0; i < 5; i++ {
        order := Order{
            OrderID: i + 1,
            Product: "Widget",
            Qty:     i * 10,
        }

        body, _ := json.Marshal(order)

        err := conn.Send("/queue/orders", "application/json", body,
            stomp.SendOpt.Header("custom-header", "custom-value"),
        )
        if err != nil {
            log.Printf("Ошибка отправки: %v", err)
        } else {
            fmt.Printf("Отправлено: %s\n", string(body))
        }

        time.Sleep(time.Second)
    }

    // Ожидание
    time.Sleep(5 * time.Second)
}
```

### Java (Spring Framework)

```java
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.simp.stomp.*;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;

import java.lang.reflect.Type;
import java.util.concurrent.ExecutionException;

public class StompClientExample {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        WebSocketStompClient stompClient = new WebSocketStompClient(
            new StandardWebSocketClient()
        );
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());

        StompSessionHandler sessionHandler = new StompSessionHandlerAdapter() {
            @Override
            public void afterConnected(StompSession session, StompHeaders headers) {
                System.out.println("Подключено к " + headers.getFirst("server"));

                // Подписка
                session.subscribe("/queue/orders", new StompFrameHandler() {
                    @Override
                    public Type getPayloadType(StompHeaders headers) {
                        return String.class;
                    }

                    @Override
                    public void handleFrame(StompHeaders headers, Object payload) {
                        System.out.println("Получено: " + payload);
                    }
                });

                // Отправка
                session.send("/queue/orders", "{\"message\": \"Hello STOMP!\"}");
            }

            @Override
            public void handleException(StompSession session, StompCommand command,
                                         StompHeaders headers, byte[] payload, Throwable exception) {
                System.err.println("Ошибка: " + exception.getMessage());
            }
        };

        StompSession session = stompClient.connect(
            "ws://localhost:15674/ws",
            sessionHandler
        ).get();

        // Держим соединение открытым
        Thread.sleep(60000);
    }
}
```

## Режимы ACK

### auto

Сообщения подтверждаются автоматически при доставке:

```python
conn.subscribe(destination='/queue/test', id='sub-1', ack='auto')
```

### client

Клиент должен подтвердить все сообщения до включительно:

```python
conn.subscribe(destination='/queue/test', id='sub-1', ack='client')

# Подтверждает это сообщение и все предыдущие
conn.ack(message_id, subscription_id)
```

### client-individual

Клиент подтверждает каждое сообщение индивидуально:

```python
conn.subscribe(destination='/queue/test', id='sub-1', ack='client-individual')

# Подтверждает только это сообщение
conn.ack(message_id, subscription_id)

# Отклонение сообщения
conn.nack(message_id, subscription_id)
```

## Receipts (Квитанции)

Receipts позволяют убедиться, что сервер обработал команду:

```python
import stomp
import threading

receipt_received = threading.Event()

class ReceiptListener(stomp.ConnectionListener):
    def on_receipt(self, frame):
        print(f"Квитанция получена: {frame.headers['receipt-id']}")
        receipt_received.set()

conn = stomp.Connection([('localhost', 61613)])
conn.set_listener('', ReceiptListener())
conn.connect('guest', 'guest', wait=True)

# Отправка с запросом квитанции
conn.send(
    destination='/queue/orders',
    body='Important message',
    headers={'receipt': 'msg-receipt-1'}
)

# Ожидание квитанции
if receipt_received.wait(timeout=5):
    print("Сообщение доставлено")
else:
    print("Таймаут ожидания квитанции")
```

## Heartbeats

Heartbeats поддерживают соединение активным:

```
CONNECT
accept-version:1.2
heart-beat:10000,10000    <- клиент: отправка каждые 10с, ожидание каждые 10с

CONNECTED
version:1.2
heart-beat:10000,10000    <- сервер: аналогично
```

Формат: `heart-beat:<outgoing>,<incoming>` (в миллисекундах)

## Когда использовать STOMP

### Идеальные сценарии

- **Web-приложения**: Чат, уведомления в браузере
- **WebSocket интеграция**: Real-time обновления
- **Отладка**: Когда нужно видеть сообщения в чистом виде
- **Простые интеграции**: Когда не нужна сложная маршрутизация
- **Polyglot окружения**: Легко реализовать на любом языке

### Не подходит для

- Высокопроизводительные системы (overhead текстового формата)
- Сложная маршрутизация сообщений
- IoT устройства с ограниченными ресурсами
- Бинарные данные большого объёма

## Порты

| Порт | Назначение |
|------|------------|
| 61613 | STOMP без шифрования |
| 61614 | STOMP с TLS/SSL |
| 15674 | STOMP over WebSocket (RabbitMQ) |

## Отладка с telnet

Одно из преимуществ STOMP — возможность отладки через telnet:

```bash
telnet localhost 61613

# Подключение
CONNECT
accept-version:1.2
login:guest
passcode:guest

^@

# Ответ: CONNECTED

# Подписка
SUBSCRIBE
destination:/queue/test
id:0
ack:auto

^@

# Отправка
SEND
destination:/queue/test
content-type:text/plain

Hello, World!^@

# Отключение
DISCONNECT
receipt:bye

^@
```

## Мониторинг

```bash
# Просмотр STOMP подключений
rabbitmqctl list_connections protocol

# Просмотр подписок (через очереди)
rabbitmqctl list_queues name messages consumers

# Список consumers
rabbitmqctl list_consumers
```

## Заключение

STOMP — это простой и удобный текстовый протокол, идеально подходящий для веб-приложений и сценариев, где важна простота отладки и интеграции. RabbitMQ через плагины `rabbitmq_stomp` и `rabbitmq_web_stomp` обеспечивает полную поддержку STOMP, включая WebSocket для браузерных приложений. Выбирайте STOMP, когда работаете с веб-интерфейсами, нужна простая интеграция или требуется человекочитаемый протокол для отладки.

---

[prev: 03-mqtt](./03-mqtt.md) | [next: 01-what-is-kafka](../../02-kafka/01-introduction/01-what-is-kafka.md)
