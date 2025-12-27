# Продвинутые структуры данных Redis

Помимо базовых типов данных (String, List, Set, Hash, Sorted Set), Redis предоставляет продвинутые структуры данных для решения специфических задач: потоковая обработка данных, вероятностный подсчёт, работа с битами, геолокация и публикация/подписка.

---

## 1. Streams (Потоки)

### Что такое Redis Streams

Redis Streams — это append-only структура данных, представляющая собой журнал сообщений (log). Каждое сообщение имеет уникальный ID и набор полей. Streams идеально подходят для:
- Журналирования событий
- Построения очередей сообщений
- Event sourcing

### Структура ID

ID сообщения имеет формат `<millisecondsTime>-<sequenceNumber>`:
```
1609459200000-0
1609459200000-1
```

### Основные команды

#### XADD — добавление сообщения

```redis
# Добавить сообщение с автоматическим ID
XADD mystream * user alice action login ip 192.168.1.1

# Добавить с явным ID
XADD mystream 1609459200000-0 user bob action logout

# Ограничить размер потока (макс. 1000 записей)
XADD mystream MAXLEN ~ 1000 * event purchase amount 100
```

#### XLEN — количество сообщений

```redis
XLEN mystream
# (integer) 3
```

#### XRANGE — чтение диапазона

```redis
# Все сообщения
XRANGE mystream - +

# Первые 10 сообщений
XRANGE mystream - + COUNT 10

# Сообщения за определённый период
XRANGE mystream 1609459200000-0 1609459300000-0
```

#### XREAD — чтение новых сообщений

```redis
# Прочитать до 5 сообщений начиная с ID
XREAD COUNT 5 STREAMS mystream 0-0

# Блокирующее чтение (ждать новые сообщения)
XREAD BLOCK 5000 STREAMS mystream $
```

### Consumer Groups (Группы потребителей)

Consumer Groups позволяют нескольким потребителям обрабатывать сообщения из одного потока параллельно, гарантируя, что каждое сообщение обработано только один раз.

#### XGROUP — управление группами

```redis
# Создать группу, начиная с начала потока
XGROUP CREATE mystream mygroup 0 MKSTREAM

# Создать группу, читая только новые сообщения
XGROUP CREATE mystream mygroup $ MKSTREAM

# Удалить группу
XGROUP DESTROY mystream mygroup

# Информация о группах
XINFO GROUPS mystream
```

#### XREADGROUP — чтение от имени группы

```redis
# Consumer "worker1" читает до 10 непрочитанных сообщений
XREADGROUP GROUP mygroup worker1 COUNT 10 STREAMS mystream >

# Блокирующее чтение
XREADGROUP GROUP mygroup worker1 BLOCK 5000 STREAMS mystream >

# Получить pending-сообщения (ранее полученные, но не подтверждённые)
XREADGROUP GROUP mygroup worker1 STREAMS mystream 0
```

#### XACK — подтверждение обработки

```redis
# Подтвердить обработку сообщения
XACK mystream mygroup 1609459200000-0

# Подтвердить несколько сообщений
XACK mystream mygroup 1609459200000-0 1609459200000-1 1609459200000-2
```

#### XPENDING — информация о необработанных сообщениях

```redis
# Общая статистика pending сообщений
XPENDING mystream mygroup

# Детальная информация (первые 10)
XPENDING mystream mygroup - + 10
```

#### XCLAIM — перехват "зависших" сообщений

```redis
# Забрать сообщения, которые не обрабатывались более 60 секунд
XCLAIM mystream mygroup worker2 60000 1609459200000-0
```

### Пример использования Streams

```python
import redis

r = redis.Redis()

# Producer: добавляем события
r.xadd('orders', {'order_id': '12345', 'product': 'laptop', 'price': '999'})
r.xadd('orders', {'order_id': '12346', 'product': 'phone', 'price': '599'})

# Создаём consumer group
try:
    r.xgroup_create('orders', 'order_processors', id='0', mkstream=True)
except redis.exceptions.ResponseError:
    pass  # Группа уже существует

# Consumer: обрабатываем заказы
while True:
    messages = r.xreadgroup(
        groupname='order_processors',
        consumername='worker1',
        streams={'orders': '>'},
        count=10,
        block=5000
    )

    for stream, entries in messages:
        for entry_id, fields in entries:
            print(f"Processing order: {fields}")
            # Обработка заказа...
            r.xack('orders', 'order_processors', entry_id)
```

### Use Cases для Streams

- **Журналы событий**: логирование действий пользователей, аудит
- **Message Queue**: альтернатива RabbitMQ/Kafka для небольших нагрузок
- **Event Sourcing**: хранение истории изменений состояния
- **Real-time аналитика**: обработка потока данных

---

## 2. HyperLogLog

### Что такое HyperLogLog

HyperLogLog — это вероятностная структура данных для подсчёта количества уникальных элементов (cardinality). Особенности:

- **Константная память**: ~12 KB независимо от количества элементов
- **Вероятностный подсчёт**: погрешность ~0.81%
- **Высокая производительность**: O(1) для добавления и подсчёта

### Основные команды

#### PFADD — добавление элементов

```redis
# Добавить элементы
PFADD visitors user1 user2 user3
# (integer) 1 — если структура изменилась

PFADD visitors user1 user4
# (integer) 1 — user4 новый

PFADD visitors user1 user2
# (integer) 0 — все элементы уже есть
```

#### PFCOUNT — подсчёт уникальных элементов

```redis
PFCOUNT visitors
# (integer) 4

# Подсчёт объединения нескольких HyperLogLog
PFADD page1_visitors alice bob charlie
PFADD page2_visitors bob david eve

PFCOUNT page1_visitors page2_visitors
# (integer) 5 — alice, bob, charlie, david, eve
```

#### PFMERGE — объединение структур

```redis
# Объединить page1_visitors и page2_visitors в all_visitors
PFMERGE all_visitors page1_visitors page2_visitors

PFCOUNT all_visitors
# (integer) 5
```

### Пример использования

```python
import redis
from datetime import datetime

r = redis.Redis()

def track_visitor(user_id: str):
    """Отслеживание уникальных посетителей по дням"""
    today = datetime.now().strftime('%Y-%m-%d')
    key = f"visitors:{today}"
    r.pfadd(key, user_id)
    # Устанавливаем TTL 7 дней
    r.expire(key, 7 * 24 * 60 * 60)

def get_daily_unique_visitors(date: str) -> int:
    """Получить количество уникальных посетителей за день"""
    return r.pfcount(f"visitors:{date}")

def get_weekly_unique_visitors() -> int:
    """Получить количество уникальных посетителей за неделю"""
    from datetime import timedelta
    keys = []
    for i in range(7):
        date = (datetime.now() - timedelta(days=i)).strftime('%Y-%m-%d')
        keys.append(f"visitors:{date}")
    return r.pfcount(*keys)

# Использование
track_visitor("user_123")
track_visitor("user_456")
track_visitor("user_123")  # Дубликат не повлияет на подсчёт

print(f"Уникальных сегодня: {get_daily_unique_visitors(datetime.now().strftime('%Y-%m-%d'))}")
```

### Use Cases для HyperLogLog

- **Подсчёт уникальных посетителей**: DAU, MAU
- **Уникальные IP-адреса**: мониторинг сетевого трафика
- **Уникальные запросы**: аналитика API
- **Уникальные слова/термы**: NLP задачи

### Сравнение с Set

| Метрика | Set | HyperLogLog |
|---------|-----|-------------|
| Память для 1M элементов | ~64 MB | 12 KB |
| Точность | 100% | ~99.19% |
| Получение элементов | Да | Нет |
| Удаление элементов | Да | Нет |

---

## 3. Bitmaps (Битовые карты)

### Что такое Bitmaps

Bitmaps — это не отдельный тип данных, а набор битовых операций над строками. Каждый бит может быть 0 или 1, что позволяет эффективно хранить булевы флаги.

### Основные команды

#### SETBIT / GETBIT — установка и чтение бита

```redis
# Установить бит в позиции 7 в значение 1
SETBIT user:1000:logins 7 1

# Прочитать бит
GETBIT user:1000:logins 7
# (integer) 1

GETBIT user:1000:logins 100
# (integer) 0 — по умолчанию
```

#### BITCOUNT — подсчёт установленных битов

```redis
# Подсчитать все единичные биты
BITCOUNT user:1000:logins
# (integer) 1

# Подсчитать биты в диапазоне байтов
BITCOUNT user:1000:logins 0 10
```

#### BITPOS — найти позицию первого бита

```redis
# Найти позицию первого бита со значением 1
BITPOS user:1000:logins 1

# Найти позицию первого бита со значением 0
BITPOS user:1000:logins 0
```

#### BITOP — побитовые операции

```redis
# AND, OR, XOR, NOT между битовыми строками
SETBIT monday_active 100 1
SETBIT monday_active 200 1
SETBIT tuesday_active 100 1
SETBIT tuesday_active 300 1

# Пользователи, активные оба дня
BITOP AND both_days monday_active tuesday_active
GETBIT both_days 100  # 1 — user 100 был активен оба дня
GETBIT both_days 200  # 0 — user 200 был только в понедельник

# Пользователи, активные хотя бы один день
BITOP OR any_day monday_active tuesday_active
```

#### BITFIELD — продвинутые операции

```redis
# Работа с целыми числами внутри битовой строки
BITFIELD mykey SET u8 0 200      # Установить 8-битное unsigned в позиции 0
BITFIELD mykey GET u8 0          # Прочитать
BITFIELD mykey INCRBY u8 0 10    # Инкрементировать
```

### Пример: отслеживание активности пользователей

```python
import redis
from datetime import datetime

r = redis.Redis()

def mark_user_active(user_id: int, date: str = None):
    """Отметить активность пользователя за день"""
    if date is None:
        date = datetime.now().strftime('%Y-%m-%d')
    r.setbit(f"active:{date}", user_id, 1)

def is_user_active(user_id: int, date: str) -> bool:
    """Проверить, был ли пользователь активен"""
    return bool(r.getbit(f"active:{date}", user_id))

def count_active_users(date: str) -> int:
    """Подсчитать активных пользователей за день"""
    return r.bitcount(f"active:{date}")

def get_active_streak(user_id: int, days: int = 7) -> int:
    """Подсчитать streak активности пользователя"""
    from datetime import timedelta
    streak = 0
    for i in range(days):
        date = (datetime.now() - timedelta(days=i)).strftime('%Y-%m-%d')
        if is_user_active(user_id, date):
            streak += 1
        else:
            break
    return streak

# Отметить активность
mark_user_active(1000)
mark_user_active(1001)
mark_user_active(1002)

print(f"Активных сегодня: {count_active_users(datetime.now().strftime('%Y-%m-%d'))}")
```

### Use Cases для Bitmaps

- **Отслеживание активности**: ежедневная активность пользователей
- **Feature flags**: включённые функции для пользователей
- **Разрешения**: права доступа
- **Bloom filters**: вероятностные проверки принадлежности

### Расчёт памяти

Для хранения флагов 10 миллионов пользователей:
- Bitmap: 10,000,000 бит = ~1.2 MB
- Set строк: ~80 MB (в среднем 8 байт на ID)

---

## 4. Geospatial (Геоданные)

### Что такое Geospatial

Redis Geospatial позволяет хранить координаты (долгота, широта) и выполнять геопространственные запросы: поиск ближайших точек, расчёт расстояний.

Под капотом используется Sorted Set, где score — это geohash координат.

### Основные команды

#### GEOADD — добавление точек

```redis
# GEOADD key longitude latitude member [longitude latitude member ...]
GEOADD restaurants -73.9857 40.7484 "Central Park Cafe"
GEOADD restaurants -73.9712 40.7831 "Upper East Side Diner"
GEOADD restaurants -74.0060 40.7128 "Downtown Pizza"

# Добавить несколько точек
GEOADD stores:nyc \
    -73.9857 40.7484 "store1" \
    -73.9712 40.7831 "store2" \
    -74.0060 40.7128 "store3"
```

#### GEOPOS — получение координат

```redis
GEOPOS restaurants "Central Park Cafe"
# 1) 1) "-73.98569972"
#    2) "40.74839894"

GEOPOS restaurants "Central Park Cafe" "Downtown Pizza"
```

#### GEODIST — расстояние между точками

```redis
# Расстояние в метрах (по умолчанию)
GEODIST restaurants "Central Park Cafe" "Downtown Pizza"
# "4533.4321"

# Расстояние в километрах
GEODIST restaurants "Central Park Cafe" "Downtown Pizza" km
# "4.5334"

# Доступные единицы: m, km, mi, ft
```

#### GEORADIUS / GEOSEARCH — поиск в радиусе

```redis
# Поиск в радиусе от координат (GEORADIUS устарел, используйте GEOSEARCH)
GEOSEARCH restaurants FROMMEMBER "Central Park Cafe" BYRADIUS 5 km
# 1) "Central Park Cafe"
# 2) "Upper East Side Diner"

# С координатами и расстоянием
GEOSEARCH restaurants FROMMEMBER "Central Park Cafe" BYRADIUS 5 km WITHDIST WITHCOORD
# 1) 1) "Central Park Cafe"
#    2) "0.0000"
#    3) 1) "-73.98569972"
#       2) "40.74839894"

# Поиск от произвольных координат
GEOSEARCH restaurants FROMLONLAT -73.9857 40.7484 BYRADIUS 10 km COUNT 5 ASC

# Поиск в прямоугольной области
GEOSEARCH restaurants FROMLONLAT -73.9857 40.7484 BYBOX 10 10 km
```

#### GEOHASH — получение geohash

```redis
GEOHASH restaurants "Central Park Cafe"
# 1) "dr5rugp8p00"
```

### Пример: поиск ближайших ресторанов

```python
import redis

r = redis.Redis(decode_responses=True)

def add_restaurant(name: str, longitude: float, latitude: float, cuisine: str = None):
    """Добавить ресторан"""
    r.geoadd("restaurants", (longitude, latitude, name))
    if cuisine:
        r.hset(f"restaurant:{name}", mapping={
            "cuisine": cuisine,
            "longitude": longitude,
            "latitude": latitude
        })

def find_nearby_restaurants(longitude: float, latitude: float, radius_km: float, limit: int = 10):
    """Найти ближайшие рестораны"""
    results = r.geosearch(
        "restaurants",
        longitude=longitude,
        latitude=latitude,
        radius=radius_km,
        unit="km",
        withdist=True,
        sort="ASC",
        count=limit
    )

    restaurants = []
    for name, distance in results:
        info = r.hgetall(f"restaurant:{name}")
        restaurants.append({
            "name": name,
            "distance_km": float(distance),
            "cuisine": info.get("cuisine", "unknown")
        })
    return restaurants

# Добавляем рестораны
add_restaurant("Pasta House", -73.9857, 40.7484, "Italian")
add_restaurant("Sushi Bar", -73.9712, 40.7831, "Japanese")
add_restaurant("Burger Joint", -74.0060, 40.7128, "American")

# Ищем ближайшие
user_location = (-73.98, 40.75)
nearby = find_nearby_restaurants(*user_location, radius_km=5)

for r in nearby:
    print(f"{r['name']} ({r['cuisine']}): {r['distance_km']:.2f} km")
```

### Use Cases для Geospatial

- **Поиск ближайших**: рестораны, магазины, такси
- **Геозоны**: определение нахождения в зоне
- **Доставка**: расчёт расстояний, оптимизация маршрутов
- **Социальные сети**: друзья поблизости

---

## 5. Pub/Sub (Публикация/Подписка)

### Что такое Pub/Sub

Pub/Sub — это механизм асинхронного обмена сообщениями между издателями и подписчиками. Особенности:
- **Fire and forget**: сообщения не сохраняются
- **В реальном времени**: доставка без задержек
- **Многие ко многим**: множество издателей и подписчиков

### Основные команды

#### SUBSCRIBE — подписка на каналы

```redis
# Подписаться на канал
SUBSCRIBE news

# Подписаться на несколько каналов
SUBSCRIBE news sports weather
```

#### PUBLISH — публикация сообщения

```redis
# Опубликовать сообщение в канал
PUBLISH news "Breaking: Redis 8.0 released!"
# (integer) 2 — количество получателей

PUBLISH sports "Goal scored!"
```

#### PSUBSCRIBE — подписка по паттерну

```redis
# Подписаться на все каналы, начинающиеся с "news."
PSUBSCRIBE news.*

# Подписаться на все каналы
PSUBSCRIBE *
```

#### UNSUBSCRIBE / PUNSUBSCRIBE — отписка

```redis
UNSUBSCRIBE news
PUNSUBSCRIBE news.*
```

### Пример: система уведомлений

```python
import redis
import threading

r = redis.Redis()

# Publisher
def publish_notification(channel: str, message: str):
    """Отправить уведомление"""
    r.publish(channel, message)

# Subscriber
def notification_handler():
    """Обработчик уведомлений"""
    pubsub = r.pubsub()
    pubsub.subscribe("notifications:user:1000")
    pubsub.psubscribe("broadcast:*")

    for message in pubsub.listen():
        if message['type'] in ('message', 'pmessage'):
            channel = message.get('channel', message.get('pattern'))
            data = message['data']
            if isinstance(data, bytes):
                data = data.decode('utf-8')
            print(f"[{channel}] {data}")

# Запуск подписчика в отдельном потоке
subscriber_thread = threading.Thread(target=notification_handler, daemon=True)
subscriber_thread.start()

# Отправка уведомлений
publish_notification("notifications:user:1000", "You have a new message!")
publish_notification("broadcast:system", "System maintenance at 3 AM")
```

### Пример: чат-комната

```python
import redis
import threading

class ChatRoom:
    def __init__(self, room_name: str):
        self.room_name = room_name
        self.redis = redis.Redis()
        self.pubsub = self.redis.pubsub()
        self.channel = f"chat:{room_name}"

    def join(self, username: str, on_message):
        """Присоединиться к чату"""
        self.username = username
        self.pubsub.subscribe(**{self.channel: on_message})
        self.thread = self.pubsub.run_in_thread(sleep_time=0.001)
        self.send(f"{username} joined the room")

    def send(self, message: str):
        """Отправить сообщение"""
        formatted = f"{self.username}: {message}"
        self.redis.publish(self.channel, formatted)

    def leave(self):
        """Покинуть чат"""
        self.send(f"{self.username} left the room")
        self.thread.stop()
        self.pubsub.unsubscribe(self.channel)

# Использование
def print_message(message):
    if message['type'] == 'message':
        print(message['data'].decode('utf-8'))

room = ChatRoom("general")
room.join("Alice", print_message)
room.send("Hello everyone!")
```

### Ограничения Pub/Sub

1. **Нет персистентности**: если подписчик офлайн — сообщение потеряно
2. **Нет acknowledgment**: нет подтверждения получения
3. **Нет истории**: нельзя получить старые сообщения

Для надёжной доставки используйте **Redis Streams**.

### Use Cases для Pub/Sub

- **Real-time уведомления**: push-сообщения
- **Чаты и мессенджеры**: мгновенный обмен сообщениями
- **Инвалидация кэша**: уведомление о изменениях
- **Мониторинг**: трансляция метрик в реальном времени
- **WebSocket broadcast**: рассылка событий клиентам

---

## Сравнение структур данных

| Структура | Персистентность | Точность | Память | Use Case |
|-----------|-----------------|----------|--------|----------|
| Streams | Да | 100% | Высокая | Event log, очереди |
| HyperLogLog | Да | ~99% | 12 KB | Подсчёт уникальных |
| Bitmaps | Да | 100% | Низкая | Флаги, активность |
| Geospatial | Да | 100% | Средняя | Геолокация |
| Pub/Sub | Нет | N/A | N/A | Real-time messaging |

---

## Резюме

Redis предоставляет мощные продвинутые структуры данных:

1. **Streams** — для надёжных очередей сообщений и event sourcing
2. **HyperLogLog** — для подсчёта уникальных элементов с минимальным потреблением памяти
3. **Bitmaps** — для эффективного хранения булевых флагов
4. **Geospatial** — для геолокационных запросов
5. **Pub/Sub** — для real-time коммуникации

Выбор структуры зависит от требований к точности, персистентности и характера задачи.
