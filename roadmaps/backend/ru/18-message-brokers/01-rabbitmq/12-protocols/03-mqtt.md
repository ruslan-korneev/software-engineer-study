# MQTT

[prev: 02-amqp-10](./02-amqp-10.md) | [next: 04-stomp](./04-stomp.md)

---

## Введение

**MQTT** (Message Queuing Telemetry Transport) — это легковесный протокол обмена сообщениями, разработанный для устройств с ограниченными ресурсами и нестабильных сетей. Изначально создан для IoT (Internet of Things) и M2M (Machine-to-Machine) коммуникаций. RabbitMQ поддерживает MQTT через плагин `rabbitmq_mqtt`.

## Характеристики MQTT

### Преимущества протокола

| Характеристика | Описание |
|----------------|----------|
| **Легковесность** | Минимальный overhead, маленький размер пакетов |
| **Publish/Subscribe** | Простая модель публикации/подписки |
| **QoS уровни** | 3 уровня гарантии доставки |
| **Retained Messages** | Сохранение последнего сообщения |
| **Last Will** | Уведомление о disconnection |
| **Keep Alive** | Поддержание соединения |

### Сравнение с AMQP

| Аспект | MQTT | AMQP 0-9-1 |
|--------|------|------------|
| Размер заголовка | 2 байта минимум | Больше |
| Модель | Только Pub/Sub | Pub/Sub + Point-to-Point |
| Маршрутизация | Только topics | Exchanges, bindings |
| Сложность | Простой | Сложный |
| Целевое применение | IoT, мобильные | Enterprise, backends |

## Архитектура MQTT

### Модель Publish/Subscribe

```
Publisher ────[Topic]────> Broker ────[Topic]────> Subscriber(s)

Пример:
Sensor ──[sensors/temp/room1]──> RabbitMQ ──[sensors/temp/#]──> Dashboard
```

### Topics (Темы)

Topics в MQTT — это иерархические строки, разделённые `/`:

```
sensors/temperature/room1
sensors/temperature/room2
sensors/humidity/room1
home/livingroom/light
home/kitchen/temperature
```

### Wildcards (Подстановочные символы)

| Символ | Описание | Пример |
|--------|----------|--------|
| `+` | Один уровень | `sensors/+/room1` → sensors/temperature/room1, sensors/humidity/room1 |
| `#` | Все уровни (только в конце) | `sensors/#` → sensors/temperature/room1, sensors/humidity/room2 |

## QoS Levels (Уровни качества обслуживания)

### QoS 0: At most once

```
Publisher ────[PUBLISH]────> Broker
          (fire and forget)
```

- Сообщение отправляется один раз
- Нет подтверждения
- Возможна потеря сообщений
- Самый быстрый

### QoS 1: At least once

```
Publisher ────[PUBLISH]────> Broker
          <────[PUBACK]────
```

- Гарантирована доставка минимум один раз
- Возможны дубликаты
- Сообщение хранится до подтверждения

### QoS 2: Exactly once

```
Publisher ────[PUBLISH]────> Broker
          <────[PUBREC]────
          ────[PUBREL]────>
          <────[PUBCOMP]───
```

- Гарантирована доставка ровно один раз
- Четырёхэтапное рукопожатие
- Самый медленный, но самый надёжный

### Выбор QoS

| Сценарий | Рекомендуемый QoS |
|----------|-------------------|
| Телеметрия датчиков (часто) | QoS 0 |
| Состояние устройств | QoS 1 |
| Финансовые транзакции | QoS 2 |
| Управление устройствами | QoS 1 или 2 |

## Установка плагина в RabbitMQ

### Включение плагина

```bash
# Включить MQTT плагин
rabbitmq-plugins enable rabbitmq_mqtt

# Опционально: включить WebSocket для MQTT
rabbitmq-plugins enable rabbitmq_web_mqtt

# Проверить статус
rabbitmq-plugins list | grep mqtt

# Перезапустить RabbitMQ
sudo systemctl restart rabbitmq-server
```

### Конфигурация

```bash
# rabbitmq.conf

# Порт для MQTT (по умолчанию 1883)
mqtt.listeners.tcp.default = 1883

# Порт для MQTT over TLS (по умолчанию 8883)
mqtt.listeners.ssl.default = 8883

# WebSocket порт (требует rabbitmq_web_mqtt)
web_mqtt.tcp.port = 15675

# Виртуальный хост по умолчанию
mqtt.vhost = /

# Пользователь по умолчанию (для анонимных подключений)
mqtt.default_user = guest
mqtt.default_pass = guest

# Разрешить анонимные подключения
mqtt.allow_anonymous = false

# Exchange для MQTT (по умолчанию amq.topic)
mqtt.exchange = amq.topic

# Prefetch count
mqtt.prefetch = 10

# Subscription TTL (в миллисекундах)
mqtt.subscription_ttl = 86400000
```

### TLS конфигурация

```bash
# rabbitmq.conf

mqtt.listeners.ssl.default = 8883
ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile = /path/to/server_certificate.pem
ssl_options.keyfile = /path/to/server_key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

## Маппинг на RabbitMQ

### Topics → Routing Keys

MQTT topics маппятся на AMQP routing keys:

| MQTT Topic | AMQP Routing Key |
|------------|------------------|
| `sensors/temp/room1` | `sensors.temp.room1` |
| `home/+/light` | `home.*.light` |
| `sensors/#` | `sensors.#` |

**Правила преобразования:**
- `/` заменяется на `.`
- `+` заменяется на `*`
- `#` остаётся `#`

### Архитектура

```
MQTT Client ──[PUBLISH sensors/temp/room1]──> RabbitMQ
                                                 │
                                                 ▼
                                          Exchange: amq.topic
                                                 │
                                    routing_key: sensors.temp.room1
                                                 │
                                                 ▼
                                          Queue (автосозданная)
                                                 │
                                                 ▼
                                          MQTT Subscriber
```

## Примеры кода

### Python (paho-mqtt)

```python
import paho.mqtt.client as mqtt
import time
import json

# Callback при подключении
def on_connect(client, userdata, flags, rc, properties=None):
    print(f"Подключено с кодом: {rc}")
    # Подписываемся на topics
    client.subscribe("sensors/temperature/#", qos=1)
    client.subscribe("sensors/humidity/+", qos=1)

# Callback при получении сообщения
def on_message(client, userdata, msg):
    print(f"Topic: {msg.topic}")
    print(f"Payload: {msg.payload.decode()}")
    print(f"QoS: {msg.qos}")
    print(f"Retained: {msg.retain}")
    print("---")

# Callback при публикации
def on_publish(client, userdata, mid, properties=None, reason_code=None):
    print(f"Сообщение {mid} опубликовано")

# Создание клиента
client = mqtt.Client(
    client_id="python-mqtt-client",
    protocol=mqtt.MQTTv5  # или MQTTv311
)

# Установка callbacks
client.on_connect = on_connect
client.on_message = on_message
client.on_publish = on_publish

# Аутентификация
client.username_pw_set("guest", "guest")

# Last Will and Testament (LWT)
client.will_set(
    topic="clients/python-client/status",
    payload="offline",
    qos=1,
    retain=True
)

# Подключение к RabbitMQ
client.connect(
    host="localhost",
    port=1883,
    keepalive=60
)

# Публикация сообщений
def publish_sensor_data():
    while True:
        # Публикация с QoS 0
        client.publish(
            topic="sensors/temperature/room1",
            payload=json.dumps({"value": 22.5, "unit": "celsius"}),
            qos=0
        )

        # Публикация с QoS 1
        result = client.publish(
            topic="sensors/humidity/room1",
            payload=json.dumps({"value": 45, "unit": "percent"}),
            qos=1
        )
        result.wait_for_publish()

        # Retained message
        client.publish(
            topic="devices/sensor1/status",
            payload="online",
            qos=1,
            retain=True
        )

        time.sleep(5)

# Запуск в фоне
client.loop_start()

try:
    publish_sensor_data()
except KeyboardInterrupt:
    # Публикуем статус перед отключением
    client.publish(
        topic="clients/python-client/status",
        payload="offline",
        qos=1,
        retain=True
    )
    client.loop_stop()
    client.disconnect()
```

### Python (Subscriber)

```python
import paho.mqtt.client as mqtt

def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        print("Успешное подключение")
        # Подписка на все датчики температуры
        client.subscribe("sensors/temperature/#", qos=1)
    else:
        print(f"Ошибка подключения: {rc}")

def on_message(client, userdata, msg):
    print(f"[{msg.topic}] {msg.payload.decode()}")

def on_disconnect(client, userdata, rc, properties=None, reason_code=None):
    print(f"Отключено: {rc}")

client = mqtt.Client(client_id="temperature-subscriber")
client.username_pw_set("guest", "guest")
client.on_connect = on_connect
client.on_message = on_message
client.on_disconnect = on_disconnect

client.connect("localhost", 1883, 60)
client.loop_forever()
```

### Node.js (mqtt.js)

```javascript
const mqtt = require('mqtt');

// Опции подключения
const options = {
    clientId: 'nodejs-mqtt-client',
    username: 'guest',
    password: 'guest',
    clean: true,
    keepalive: 60,
    reconnectPeriod: 1000,
    will: {
        topic: 'clients/nodejs-client/status',
        payload: 'offline',
        qos: 1,
        retain: true
    }
};

// Подключение
const client = mqtt.connect('mqtt://localhost:1883', options);

client.on('connect', () => {
    console.log('Подключено к MQTT брокеру');

    // Подписка
    client.subscribe('sensors/#', { qos: 1 }, (err) => {
        if (!err) {
            console.log('Подписка на sensors/#');
        }
    });

    // Публикация
    setInterval(() => {
        const payload = JSON.stringify({
            temperature: Math.random() * 30 + 10,
            timestamp: Date.now()
        });

        client.publish('sensors/temperature/room1', payload, {
            qos: 1,
            retain: false
        });
    }, 5000);
});

client.on('message', (topic, message) => {
    console.log(`Topic: ${topic}`);
    console.log(`Message: ${message.toString()}`);
});

client.on('error', (err) => {
    console.error('Ошибка:', err);
});

client.on('close', () => {
    console.log('Соединение закрыто');
});
```

### Go (eclipse/paho.mqtt.golang)

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"

    mqtt "github.com/eclipse/paho.mqtt.golang"
)

type SensorData struct {
    Value     float64 `json:"value"`
    Unit      string  `json:"unit"`
    Timestamp int64   `json:"timestamp"`
}

var messagePubHandler mqtt.MessageHandler = func(client mqtt.Client, msg mqtt.Message) {
    fmt.Printf("Topic: %s\n", msg.Topic())
    fmt.Printf("Payload: %s\n", msg.Payload())
}

var connectHandler mqtt.OnConnectHandler = func(client mqtt.Client) {
    fmt.Println("Подключено")

    // Подписка после подключения
    token := client.Subscribe("sensors/#", 1, nil)
    token.Wait()
    fmt.Println("Подписка на sensors/#")
}

var connectionLostHandler mqtt.ConnectionLostHandler = func(client mqtt.Client, err error) {
    fmt.Printf("Соединение потеряно: %v\n", err)
}

func main() {
    opts := mqtt.NewClientOptions()
    opts.AddBroker("tcp://localhost:1883")
    opts.SetClientID("go-mqtt-client")
    opts.SetUsername("guest")
    opts.SetPassword("guest")
    opts.SetKeepAlive(60 * time.Second)
    opts.SetDefaultPublishHandler(messagePubHandler)
    opts.OnConnect = connectHandler
    opts.OnConnectionLost = connectionLostHandler

    // Last Will
    opts.SetWill("clients/go-client/status", "offline", 1, true)

    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        panic(token.Error())
    }

    // Публикация
    go func() {
        for {
            data := SensorData{
                Value:     22.5,
                Unit:      "celsius",
                Timestamp: time.Now().Unix(),
            }
            payload, _ := json.Marshal(data)

            token := client.Publish("sensors/temperature/room1", 1, false, payload)
            token.Wait()
            fmt.Println("Опубликовано")

            time.Sleep(5 * time.Second)
        }
    }()

    // Держим программу запущенной
    select {}
}
```

### JavaScript (Browser WebSocket)

```html
<!DOCTYPE html>
<html>
<head>
    <title>MQTT WebSocket Client</title>
    <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
</head>
<body>
    <h1>MQTT Dashboard</h1>
    <div id="messages"></div>

    <script>
        // Подключение через WebSocket
        const client = mqtt.connect('ws://localhost:15675/ws', {
            clientId: 'browser-client-' + Math.random().toString(16).substr(2, 8),
            username: 'guest',
            password: 'guest'
        });

        client.on('connect', function () {
            console.log('Подключено');
            client.subscribe('sensors/#');
        });

        client.on('message', function (topic, message) {
            const div = document.getElementById('messages');
            const p = document.createElement('p');
            p.textContent = `${topic}: ${message.toString()}`;
            div.prepend(p);
        });

        // Публикация
        function publish() {
            client.publish('sensors/browser/click', JSON.stringify({
                timestamp: Date.now()
            }));
        }
    </script>

    <button onclick="publish()">Отправить событие</button>
</body>
</html>
```

## Retained Messages

Retained сообщения сохраняются брокером и отправляются новым подписчикам:

```python
# Публикация retained сообщения
client.publish(
    topic="devices/sensor1/status",
    payload="online",
    qos=1,
    retain=True  # Сообщение будет сохранено
)

# Очистка retained сообщения
client.publish(
    topic="devices/sensor1/status",
    payload="",  # Пустое сообщение
    qos=1,
    retain=True
)
```

## Last Will and Testament (LWT)

LWT — сообщение, которое брокер опубликует, если клиент неожиданно отключится:

```python
client.will_set(
    topic="clients/my-client/status",
    payload="unexpected_disconnect",
    qos=1,
    retain=True
)
```

## Clean Session

```python
# Clean Session = True (по умолчанию)
# - Все подписки удаляются при отключении
# - Неполученные сообщения теряются
client = mqtt.Client(client_id="client1", clean_session=True)

# Clean Session = False
# - Подписки сохраняются
# - QoS 1/2 сообщения накапливаются
client = mqtt.Client(client_id="client1", clean_session=False)
```

## MQTT 5.0 Features

MQTT 5.0 добавляет новые возможности:

```python
from paho.mqtt.properties import Properties
from paho.mqtt.packettypes import PacketTypes

# Создание клиента MQTT 5.0
client = mqtt.Client(protocol=mqtt.MQTTv5)

# Message Expiry Interval
properties = Properties(PacketTypes.PUBLISH)
properties.MessageExpiryInterval = 60  # секунды

client.publish(
    topic="sensors/temp",
    payload="data",
    qos=1,
    properties=properties
)

# Response Topic (для Request/Response)
properties = Properties(PacketTypes.PUBLISH)
properties.ResponseTopic = "response/client1"
properties.CorrelationData = b"request-123"
```

## Когда использовать MQTT

### Идеальные сценарии

- **IoT устройства**: Датчики, актуаторы, умный дом
- **Мобильные приложения**: Уведомления, синхронизация
- **Нестабильные сети**: Мобильный интернет, спутник
- **Ограниченные ресурсы**: Встраиваемые системы
- **Real-time данные**: Телеметрия, мониторинг

### Не подходит для

- Сложная маршрутизация сообщений
- Point-to-point коммуникация
- Транзакционные системы
- Большие объёмы данных в одном сообщении

## Порты

| Порт | Назначение |
|------|------------|
| 1883 | MQTT без шифрования |
| 8883 | MQTT с TLS/SSL |
| 15675 | MQTT over WebSocket (RabbitMQ) |

## Мониторинг

```bash
# Просмотр MQTT подключений
rabbitmqctl list_connections protocol

# Просмотр MQTT подписок (через очереди)
rabbitmqctl list_queues name messages consumers

# Статистика
rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

## Заключение

MQTT — это идеальный протокол для IoT и мобильных приложений благодаря своей легковесности и простоте. RabbitMQ через плагин `rabbitmq_mqtt` позволяет интегрировать MQTT устройства с корпоративной инфраструктурой обмена сообщениями, маппируя MQTT topics на AMQP routing keys. Выбирайте MQTT, когда работаете с устройствами с ограниченными ресурсами или нестабильными сетями.

---

[prev: 02-amqp-10](./02-amqp-10.md) | [next: 04-stomp](./04-stomp.md)
