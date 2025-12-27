# Redis: Начало работы

## Установка Redis

### Linux (Ubuntu/Debian)

#### Установка из официального репозитория

```bash
# Добавление официального Redis репозитория
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Установка Redis
sudo apt-get update
sudo apt-get install redis
```

#### Установка из стандартного репозитория

```bash
sudo apt-get update
sudo apt-get install redis-server
```

#### Проверка установки

```bash
redis-server --version
# Redis server v=7.2.4 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=...
```

### macOS

#### Через Homebrew

```bash
# Установка
brew install redis

# Запуск как сервис (автозапуск при старте системы)
brew services start redis

# Или однократный запуск
redis-server
```

#### Проверка

```bash
redis-cli ping
# PONG
```

### Docker

Docker — самый простой и кроссплатформенный способ запустить Redis:

```bash
# Запуск контейнера Redis
docker run --name my-redis -d -p 6379:6379 redis

# Запуск с персистентностью (данные сохраняются в volume)
docker run --name my-redis -d -p 6379:6379 -v redis-data:/data redis redis-server --appendonly yes

# Подключение к запущенному контейнеру
docker exec -it my-redis redis-cli
```

#### Docker Compose

Для более сложных конфигураций используйте `docker-compose.yml`:

```yaml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: unless-stopped

volumes:
  redis_data:
```

Запуск:
```bash
docker-compose up -d
```

### Windows

Redis официально не поддерживает Windows, но есть варианты:

1. **WSL2 (рекомендуется)** — установка через Ubuntu в WSL2
2. **Docker Desktop** — использование Docker контейнера
3. **Memurai** — Windows-совместимый форк Redis (для продакшена)

## Запуск и управление сервером

### Запуск Redis сервера

```bash
# Запуск с настройками по умолчанию
redis-server

# Запуск с конфигурационным файлом
redis-server /path/to/redis.conf

# Запуск с параметрами командной строки
redis-server --port 6380 --requirepass mypassword
```

### Управление сервисом (systemd)

```bash
# Статус сервиса
sudo systemctl status redis

# Запуск
sudo systemctl start redis

# Остановка
sudo systemctl stop redis

# Перезапуск
sudo systemctl restart redis

# Автозапуск при старте системы
sudo systemctl enable redis
```

### Проверка работы

```bash
# Проверка, что Redis отвечает
redis-cli ping
# PONG

# Информация о сервере
redis-cli info server
```

## Redis CLI: базовые команды

`redis-cli` — это интерактивный терминал для работы с Redis.

### Подключение

```bash
# Подключение к локальному серверу
redis-cli

# Подключение к удалённому серверу
redis-cli -h hostname -p 6379

# Подключение с паролем
redis-cli -h hostname -p 6379 -a password

# Подключение к конкретной базе данных (по умолчанию 0)
redis-cli -n 1
```

### Базовые команды для работы со строками

```bash
# SET — сохранение значения
SET mykey "Hello, Redis!"
# OK

# GET — получение значения
GET mykey
# "Hello, Redis!"

# SET с опциями
SET session:123 "user_data" EX 3600       # истекает через 3600 секунд
SET counter 100 NX                         # установить только если ключ не существует
SET counter 200 XX                         # установить только если ключ существует

# MSET/MGET — множественные операции
MSET key1 "value1" key2 "value2" key3 "value3"
MGET key1 key2 key3
# 1) "value1"
# 2) "value2"
# 3) "value3"

# INCR/DECR — атомарный инкремент/декремент
SET counter 10
INCR counter      # 11
INCR counter      # 12
DECR counter      # 11
INCRBY counter 5  # 16
DECRBY counter 3  # 13
```

### Управление ключами

```bash
# DEL — удаление ключа
DEL mykey
# (integer) 1

# EXISTS — проверка существования
EXISTS mykey
# (integer) 0

# KEYS — поиск ключей по паттерну (НЕ использовать в продакшене!)
KEYS user:*
# 1) "user:1"
# 2) "user:2"

# SCAN — безопасный перебор ключей
SCAN 0 MATCH user:* COUNT 100

# TTL/PTTL — время жизни ключа
TTL mykey        # в секундах
PTTL mykey       # в миллисекундах

# EXPIRE — установка времени жизни
EXPIRE mykey 300      # истечёт через 300 секунд
EXPIREAT mykey 1735689600  # истечёт в указанный unix timestamp

# PERSIST — удаление TTL (ключ становится "вечным")
PERSIST mykey

# TYPE — тип значения
TYPE mykey
# string

# RENAME — переименование ключа
RENAME oldkey newkey
```

### Работа со списками (Lists)

```bash
# LPUSH/RPUSH — добавление элементов
LPUSH mylist "first"   # добавить в начало
RPUSH mylist "last"    # добавить в конец

# LRANGE — получение диапазона элементов
LRANGE mylist 0 -1     # все элементы
# 1) "first"
# 2) "last"

# LPOP/RPOP — извлечение элементов
LPOP mylist            # извлечь первый
RPOP mylist            # извлечь последний

# LLEN — длина списка
LLEN mylist

# BRPOP/BLPOP — блокирующее извлечение (для очередей)
BRPOP mylist 30        # ждать 30 секунд
```

### Работа с хэшами (Hashes)

```bash
# HSET — установка полей
HSET user:1 name "John" age "30" email "john@example.com"

# HGET — получение одного поля
HGET user:1 name
# "John"

# HGETALL — получение всех полей
HGETALL user:1
# 1) "name"
# 2) "John"
# 3) "age"
# 4) "30"
# 5) "email"
# 6) "john@example.com"

# HMGET — получение нескольких полей
HMGET user:1 name email

# HDEL — удаление поля
HDEL user:1 email

# HINCRBY — инкремент числового поля
HINCRBY user:1 age 1
```

### Работа с множествами (Sets)

```bash
# SADD — добавление элементов
SADD tags "python" "redis" "backend"

# SMEMBERS — все элементы
SMEMBERS tags

# SISMEMBER — проверка принадлежности
SISMEMBER tags "python"
# (integer) 1

# SCARD — количество элементов
SCARD tags

# SINTER — пересечение множеств
SINTER tags1 tags2

# SUNION — объединение множеств
SUNION tags1 tags2
```

### Работа с отсортированными множествами (Sorted Sets)

```bash
# ZADD — добавление с весом (score)
ZADD leaderboard 1500 "player1" 2300 "player2" 1800 "player3"

# ZRANGE — элементы по возрастанию score
ZRANGE leaderboard 0 -1 WITHSCORES

# ZREVRANGE — элементы по убыванию score (топ игроков)
ZREVRANGE leaderboard 0 2 WITHSCORES
# 1) "player2"
# 2) "2300"
# 3) "player3"
# 4) "1800"
# 5) "player1"
# 6) "1500"

# ZRANK/ZREVRANK — позиция в рейтинге
ZREVRANK leaderboard "player1"
# (integer) 2

# ZINCRBY — увеличение score
ZINCRBY leaderboard 100 "player1"
```

## Подключение из Python (redis-py)

### Установка

```bash
pip install redis
```

### Базовое подключение

```python
import redis

# Простое подключение
r = redis.Redis(host='localhost', port=6379, db=0)

# С декодированием ответов (рекомендуется)
r = redis.Redis(
    host='localhost',
    port=6379,
    db=0,
    decode_responses=True  # автоматическое декодирование bytes -> str
)

# Подключение с паролем
r = redis.Redis(
    host='localhost',
    port=6379,
    password='your_password',
    decode_responses=True
)

# Подключение через URL
r = redis.from_url('redis://user:password@localhost:6379/0')
```

### Базовые операции

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Строки
r.set('name', 'Alice')
r.set('counter', 0)
r.setex('temp_key', 3600, 'temporary value')  # TTL 1 час

name = r.get('name')  # 'Alice'
r.incr('counter')     # 1
r.incrby('counter', 5)  # 6

# Проверка существования и удаление
exists = r.exists('name')  # 1
r.delete('name')
exists = r.exists('name')  # 0

# Хэши
r.hset('user:1', mapping={
    'name': 'John',
    'age': '30',
    'email': 'john@example.com'
})

user = r.hgetall('user:1')
# {'name': 'John', 'age': '30', 'email': 'john@example.com'}

name = r.hget('user:1', 'name')  # 'John'

# Списки
r.rpush('tasks', 'task1', 'task2', 'task3')
tasks = r.lrange('tasks', 0, -1)  # ['task1', 'task2', 'task3']
task = r.lpop('tasks')  # 'task1'

# Множества
r.sadd('skills', 'python', 'redis', 'docker')
skills = r.smembers('skills')  # {'python', 'redis', 'docker'}
has_python = r.sismember('skills', 'python')  # True

# Отсортированные множества
r.zadd('leaderboard', {'player1': 1500, 'player2': 2300})
top_players = r.zrevrange('leaderboard', 0, 9, withscores=True)
# [('player2', 2300.0), ('player1', 1500.0)]
```

### Пайплайны (Pipeline)

Пайплайны позволяют отправить несколько команд за один раунд-трип:

```python
# Без пайплайна — N раунд-трипов для N команд
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')

# С пайплайном — 1 раунд-трип для N команд
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
```

### Пул соединений

```python
import redis

# Создание пула соединений
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=10,
    decode_responses=True
)

# Использование пула
r = redis.Redis(connection_pool=pool)
```

## Подключение из Node.js (ioredis)

### Установка

```bash
npm install ioredis
# или
yarn add ioredis
```

### Базовое подключение

```javascript
const Redis = require('ioredis');

// Простое подключение
const redis = new Redis();

// С параметрами
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'your_password',
  db: 0,
});

// Через URL
const redis = new Redis('redis://user:password@localhost:6379/0');
```

### Базовые операции

```javascript
const Redis = require('ioredis');
const redis = new Redis();

// Строки
await redis.set('name', 'Alice');
await redis.setex('temp_key', 3600, 'temporary value');

const name = await redis.get('name');  // 'Alice'
await redis.incr('counter');

// Хэши
await redis.hset('user:1', {
  name: 'John',
  age: '30',
  email: 'john@example.com'
});

const user = await redis.hgetall('user:1');
// { name: 'John', age: '30', email: 'john@example.com' }

// Списки
await redis.rpush('tasks', 'task1', 'task2', 'task3');
const tasks = await redis.lrange('tasks', 0, -1);
// ['task1', 'task2', 'task3']

// Множества
await redis.sadd('skills', 'javascript', 'redis', 'docker');
const skills = await redis.smembers('skills');

// Отсортированные множества
await redis.zadd('leaderboard', 1500, 'player1', 2300, 'player2');
const topPlayers = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
```

### Пайплайны

```javascript
const pipeline = redis.pipeline();

for (let i = 0; i < 1000; i++) {
  pipeline.set(`key:${i}`, `value:${i}`);
}

const results = await pipeline.exec();
```

### Pub/Sub

```javascript
const Redis = require('ioredis');

// Подписчик
const subscriber = new Redis();
subscriber.subscribe('notifications');

subscriber.on('message', (channel, message) => {
  console.log(`Received ${message} from ${channel}`);
});

// Издатель (отдельное соединение!)
const publisher = new Redis();
await publisher.publish('notifications', JSON.stringify({ type: 'alert', text: 'Hello!' }));
```

## Базовые операции SET/GET/DEL: сводка

| Операция | Redis CLI | Python (redis-py) | Node.js (ioredis) |
|----------|-----------|-------------------|-------------------|
| Установка | `SET key value` | `r.set('key', 'value')` | `await redis.set('key', 'value')` |
| Установка с TTL | `SETEX key 3600 value` | `r.setex('key', 3600, 'value')` | `await redis.setex('key', 3600, 'value')` |
| Получение | `GET key` | `r.get('key')` | `await redis.get('key')` |
| Удаление | `DEL key` | `r.delete('key')` | `await redis.del('key')` |
| Проверка | `EXISTS key` | `r.exists('key')` | `await redis.exists('key')` |
| Инкремент | `INCR key` | `r.incr('key')` | `await redis.incr('key')` |

## Полезные команды для отладки

```bash
# Информация о сервере
INFO

# Мониторинг команд в реальном времени
MONITOR

# Размер базы данных
DBSIZE

# Очистка текущей базы данных
FLUSHDB

# Очистка всех баз данных
FLUSHALL

# Последние логи медленных команд
SLOWLOG GET 10

# Информация о клиентах
CLIENT LIST

# Конфигурация
CONFIG GET maxmemory
CONFIG SET maxmemory 100mb
```

## Заключение

После установки Redis и изучения базовых команд вы готовы использовать его в своих проектах. Основные шаги:

1. Установите Redis (Docker — самый простой способ для разработки)
2. Научитесь работать с `redis-cli` для отладки и экспериментов
3. Подключите клиентскую библиотеку к вашему приложению
4. Используйте правильные структуры данных для ваших задач

В следующих разделах мы подробнее рассмотрим продвинутые возможности Redis: транзакции, Lua скрипты, Pub/Sub, Streams и настройку кластера.
