# Redis Modules

## Что такое Redis Modules

**Redis Modules** — расширения, добавляющие новые типы данных и команды в Redis. Модули написаны на C и загружаются как динамические библиотеки.

```redis
# Загрузка модуля
MODULE LOAD /path/to/module.so

# Список загруженных модулей
MODULE LIST
```

## Установка модулей

### Через Redis Stack

Redis Stack — дистрибутив Redis с предустановленными модулями:

```bash
# Docker
docker run -d -p 6379:6379 redis/redis-stack

# Включает: RedisJSON, RediSearch, RedisTimeSeries, RedisBloom, RedisGraph
```

### Ручная установка

```bash
# Сборка модуля из исходников
git clone https://github.com/RedisJSON/RedisJSON.git
cd RedisJSON
make

# Загрузка в redis.conf
loadmodule /path/to/rejson.so
```

## RedisJSON

Работа с JSON-документами как с нативным типом данных.

### Основные команды

```redis
# Создание JSON-документа
JSON.SET user:1 $ '{"name":"John","age":30,"email":"john@example.com"}'

# Получение всего документа
JSON.GET user:1
# {"name":"John","age":30,"email":"john@example.com"}

# Получение поля
JSON.GET user:1 $.name
# ["John"]

# Изменение поля
JSON.SET user:1 $.age 31

# Инкремент числового поля
JSON.NUMINCRBY user:1 $.age 1

# Добавление в массив
JSON.SET user:1 $.tags '[]'
JSON.ARRAPPEND user:1 $.tags '"redis"' '"developer"'
```

### JSONPath синтаксис

```redis
# Корень документа
$

# Конкретное поле
$.name

# Вложенное поле
$.address.city

# Все элементы массива
$.items[*]

# Конкретный индекс
$.items[0]

# Фильтрация
$.items[?(@.price > 100)]
```

### Практический пример

```redis
# Профиль пользователя
JSON.SET profile:1 $ '{
  "username": "john_doe",
  "stats": {"posts": 0, "followers": 0},
  "settings": {"theme": "dark", "notifications": true}
}'

# Увеличить счётчик постов
JSON.NUMINCRBY profile:1 $.stats.posts 1

# Изменить настройку
JSON.SET profile:1 $.settings.theme '"light"'

# Получить только статистику
JSON.GET profile:1 $.stats
```

## RediSearch

Полнотекстовый поиск и вторичные индексы.

### Создание индекса

```redis
# Индекс для хэшей user:*
FT.CREATE idx:users
  ON HASH PREFIX 1 user:
  SCHEMA
    name TEXT SORTABLE
    email TEXT
    age NUMERIC SORTABLE
    city TAG
```

Типы полей:
- `TEXT` — полнотекстовый поиск
- `NUMERIC` — числа (range queries)
- `TAG` — точное совпадение
- `GEO` — геолокация

### Индексация данных

```redis
# Данные автоматически индексируются при записи
HSET user:1 name "John Smith" email "john@example.com" age 30 city "NYC"
HSET user:2 name "Jane Doe" email "jane@example.com" age 25 city "LA"
HSET user:3 name "John Doe" email "jdoe@example.com" age 35 city "NYC"
```

### Поиск

```redis
# Полнотекстовый поиск
FT.SEARCH idx:users "John"
# Найдёт user:1 и user:3

# Поиск по полю
FT.SEARCH idx:users "@name:John"

# Числовой range
FT.SEARCH idx:users "@age:[25 30]"

# Точное совпадение (TAG)
FT.SEARCH idx:users "@city:{NYC}"

# Комбинированный запрос
FT.SEARCH idx:users "@name:John @city:{NYC}"

# Сортировка
FT.SEARCH idx:users "*" SORTBY age DESC

# Пагинация
FT.SEARCH idx:users "*" LIMIT 0 10
```

### Агрегации

```redis
# Средний возраст по городам
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @city
  REDUCE AVG 1 @age AS avg_age

# Количество пользователей по городам
FT.AGGREGATE idx:users "*"
  GROUPBY 1 @city
  REDUCE COUNT 0 AS count
```

## RedisTimeSeries

Хранение и анализ временных рядов.

### Создание временного ряда

```redis
# Создание с настройками
TS.CREATE sensor:temp:1
  RETENTION 86400000
  LABELS type temperature location room1

# RETENTION — время хранения в миллисекундах (24 часа)
# LABELS — метки для фильтрации
```

### Добавление данных

```redis
# Добавить точку (timestamp в мс, значение)
TS.ADD sensor:temp:1 * 23.5
# * = текущее время

# Или с явным timestamp
TS.ADD sensor:temp:1 1609459200000 23.5

# Массовое добавление
TS.MADD sensor:temp:1 * 23.5 sensor:temp:2 * 24.0
```

### Запросы

```redis
# Последнее значение
TS.GET sensor:temp:1

# Range запрос
TS.RANGE sensor:temp:1 - +
# - = минимальное время, + = максимальное время

# С агрегацией (среднее за 1 час)
TS.RANGE sensor:temp:1 - + AGGREGATION avg 3600000

# Последние N значений
TS.RANGE sensor:temp:1 - + COUNT 10

# Запрос по меткам
TS.MRANGE - + FILTER type=temperature
```

### Downsampling (автоматическая агрегация)

```redis
# Создать правило агрегации
TS.CREATERULE sensor:temp:1 sensor:temp:1:hourly
  AGGREGATION avg 3600000

# Теперь sensor:temp:1:hourly автоматически заполняется
# средними значениями за каждый час
```

## RedisBloom

Вероятностные структуры данных.

### Bloom Filter

Проверка принадлежности элемента множеству с возможными false positives.

```redis
# Создание фильтра
BF.RESERVE visited_urls 0.01 1000000
# error_rate=1%, capacity=1000000

# Добавление элементов
BF.ADD visited_urls "https://example.com/page1"
BF.ADD visited_urls "https://example.com/page2"

# Проверка
BF.EXISTS visited_urls "https://example.com/page1"
# (integer) 1 — возможно существует

BF.EXISTS visited_urls "https://example.com/unknown"
# (integer) 0 — точно не существует
```

**Use cases:**
- Проверка уникальности URL (crawler)
- Проверка, видел ли пользователь контент
- Фильтрация спама

### Cuckoo Filter

Альтернатива Bloom Filter с поддержкой удаления:

```redis
CF.RESERVE unique_items 1000000

CF.ADD unique_items "item1"
CF.EXISTS unique_items "item1"
CF.DEL unique_items "item1"  # Поддерживает удаление!
```

### Count-Min Sketch

Подсчёт частоты элементов:

```redis
# Создание
CMS.INITBYDIM page_views 2000 5

# Увеличить счётчик
CMS.INCRBY page_views "/home" 1 "/about" 1 "/home" 1

# Получить приблизительный счёт
CMS.QUERY page_views "/home"
# (integer) 2
```

**Use cases:**
- Подсчёт просмотров страниц
- Частотный анализ
- Детекция аномалий

### Top-K

Отслеживание самых частых элементов:

```redis
# Создание (топ-10)
TOPK.RESERVE trending_topics 10

# Добавление событий
TOPK.ADD trending_topics "redis" "docker" "kubernetes" "redis"

# Получить топ
TOPK.LIST trending_topics
```

### HyperLogLog (встроен в Redis)

Подсчёт уникальных элементов:

```redis
PFADD visitors:2024-01-01 "user:1" "user:2" "user:3"
PFADD visitors:2024-01-01 "user:1" "user:4"

PFCOUNT visitors:2024-01-01
# (integer) 4
```

## Выбор модуля

| Задача | Модуль |
|--------|--------|
| Хранение JSON-документов | RedisJSON |
| Полнотекстовый поиск | RediSearch |
| Метрики, IoT данные | RedisTimeSeries |
| Проверка уникальности | RedisBloom (Bloom/Cuckoo) |
| Подсчёт частоты | RedisBloom (CMS) |
| Топ элементов | RedisBloom (TopK) |
| Графы и связи | RedisGraph |

## Комбинирование модулей

RedisJSON + RediSearch — поиск по JSON:

```redis
# Индекс для JSON-документов
FT.CREATE idx:products ON JSON PREFIX 1 product: SCHEMA
  $.name AS name TEXT
  $.price AS price NUMERIC
  $.category AS category TAG

# JSON-документ
JSON.SET product:1 $ '{"name":"Redis Book","price":29.99,"category":"books"}'

# Поиск
FT.SEARCH idx:products "@category:{books} @price:[0 50]"
```
