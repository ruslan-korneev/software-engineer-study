# Pub/Sub в Redis

## Что такое Pub/Sub

**Pub/Sub (Publish/Subscribe)** — паттерн обмена сообщениями, где отправители (publishers) не знают о получателях (subscribers). Сообщения публикуются в **каналы**, а подписчики получают все сообщения из каналов, на которые подписаны.

```
Publisher 1 ──┐
              ├──> Channel "news" ──┬──> Subscriber A
Publisher 2 ──┘                     └──> Subscriber B
```

Redis реализует Pub/Sub как **fire-and-forget** систему — сообщения доставляются только подключённым подписчикам в момент публикации.

## Основные команды

### SUBSCRIBE — подписка на каналы

```redis
SUBSCRIBE channel1 channel2 channel3
```

После подписки клиент переходит в **режим подписчика** и может только:
- Подписываться на новые каналы
- Отписываться от каналов
- Получать сообщения

```redis
> SUBSCRIBE news updates
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news"
3) (integer) 1
1) "subscribe"
2) "updates"
3) (integer) 2
```

### PUBLISH — публикация сообщения

```redis
PUBLISH channel message
```

Возвращает количество подписчиков, получивших сообщение:

```redis
> PUBLISH news "Breaking: Redis 8.0 released!"
(integer) 2
```

### UNSUBSCRIBE — отписка от каналов

```redis
UNSUBSCRIBE channel1 channel2
# или отписаться от всех
UNSUBSCRIBE
```

## Pattern Subscriptions (PSUBSCRIBE)

Подписка на каналы по шаблону с использованием glob-стиля:

```redis
# Подписка на все каналы, начинающиеся с "news."
PSUBSCRIBE news.*

# Подписка на каналы вида user:123:notifications
PSUBSCRIBE user:*:notifications
```

Поддерживаемые шаблоны:
- `*` — любое количество любых символов
- `?` — ровно один любой символ
- `[abc]` — один из символов в скобках

```redis
> PSUBSCRIBE news.*
1) "psubscribe"
2) "news.*"
3) (integer) 1

# Получит сообщения из news.sports, news.politics, news.tech и т.д.
```

### PUNSUBSCRIBE — отписка по шаблону

```redis
PUNSUBSCRIBE news.*
```

## Практические примеры

### Система уведомлений

**Publisher (сервис заказов):**
```python
import redis

r = redis.Redis()

def notify_order_status(order_id, status):
    channel = f"orders:{order_id}"
    message = {"status": status, "timestamp": time.time()}
    r.publish(channel, json.dumps(message))

# Уведомление о смене статуса заказа
notify_order_status(12345, "shipped")
```

**Subscriber (клиентское приложение):**
```python
import redis

r = redis.Redis()
pubsub = r.pubsub()

# Подписка на уведомления о заказе
pubsub.subscribe("orders:12345")

for message in pubsub.listen():
    if message["type"] == "message":
        data = json.loads(message["data"])
        print(f"Order status: {data['status']}")
```

### Чат-система

```python
# Публикация сообщения в комнату
def send_message(room_id, user, text):
    r.publish(f"chat:{room_id}", json.dumps({
        "user": user,
        "text": text,
        "time": datetime.now().isoformat()
    }))

# Подписка на комнату
pubsub.psubscribe("chat:*")  # все комнаты
# или
pubsub.subscribe("chat:general")  # конкретная комната
```

### Инвалидация кэша

```python
# При изменении данных публикуем событие
def update_user(user_id, data):
    db.update_user(user_id, data)
    r.publish("cache:invalidate", f"user:{user_id}")

# Сервис кэша слушает и очищает кэш
for message in pubsub.listen():
    if message["type"] == "message":
        cache_key = message["data"].decode()
        cache.delete(cache_key)
```

## Важные ограничения

### 1. Нет гарантии доставки

Если подписчик был отключён в момент публикации — сообщение потеряно:

```
Publisher ──> "message1" ──> Subscriber (online) ✓
Publisher ──> "message2" ──> Subscriber (offline) ✗ LOST
Publisher ──> "message3" ──> Subscriber (online) ✓
```

### 2. Нет персистентности

Сообщения не сохраняются на диск. При перезапуске Redis история сообщений теряется.

### 3. Нет подтверждения получения (ACK)

Publisher не знает, обработал ли subscriber сообщение успешно.

### 4. Нет consumer groups

Если 3 подписчика слушают один канал — все трое получат одно и то же сообщение (broadcasting, не load balancing).

## Pub/Sub vs Redis Streams

| Характеристика | Pub/Sub | Streams |
|----------------|---------|---------|
| Персистентность | Нет | Да |
| История сообщений | Нет | Да |
| Consumer Groups | Нет | Да |
| Гарантия доставки | Нет | Да (с ACK) |
| Производительность | Выше | Немного ниже |
| Сложность | Простой | Сложнее |

**Когда использовать Pub/Sub:**
- Real-time уведомления, где потеря сообщений некритична
- Broadcast всем подключённым клиентам
- Простые сценарии без необходимости истории

**Когда использовать Streams:**
- Нужна надёжная доставка сообщений
- Требуется история сообщений
- Нужен load balancing между consumers
- Критичные бизнес-события

## Команды для отладки

### PUBSUB CHANNELS — список активных каналов

```redis
> PUBSUB CHANNELS
1) "news"
2) "chat:general"
3) "orders:12345"

> PUBSUB CHANNELS news*
1) "news"
```

### PUBSUB NUMSUB — количество подписчиков

```redis
> PUBSUB NUMSUB news chat:general
1) "news"
2) (integer) 5
3) "chat:general"
4) (integer) 12
```

### PUBSUB NUMPAT — количество pattern-подписок

```redis
> PUBSUB NUMPAT
(integer) 3
```

## Best Practices

1. **Не полагайтесь на доставку** — Pub/Sub для real-time, не для критичных данных
2. **Используйте namespace в именах каналов** — `service:entity:action`
3. **Мониторьте количество подписчиков** — через PUBSUB NUMSUB
4. **Ограничивайте размер сообщений** — большие сообщения замедляют всех подписчиков
5. **Обрабатывайте reconnect** — клиенты должны переподписываться при потере соединения

## Пример reconnect логики

```python
import redis
import time

def subscribe_with_reconnect(channels):
    while True:
        try:
            r = redis.Redis()
            pubsub = r.pubsub()
            pubsub.subscribe(channels)

            for message in pubsub.listen():
                if message["type"] == "message":
                    process_message(message)

        except redis.ConnectionError:
            print("Connection lost, reconnecting...")
            time.sleep(1)
```
