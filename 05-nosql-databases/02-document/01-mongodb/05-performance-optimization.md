# Оптимизация производительности MongoDB

Оптимизация производительности MongoDB — критически важный аспект работы с базой данных, особенно при масштабировании приложений. Правильная настройка индексов, схемы данных и параметров движка хранения может значительно улучшить время отклика и пропускную способность системы.

## Индексы

### Типы индексов

MongoDB поддерживает несколько типов индексов для различных сценариев использования:

**1. Single Field Index (Индекс по одному полю)**
```javascript
// Создание индекса по полю email
db.users.createIndex({ email: 1 })  // 1 = по возрастанию
db.users.createIndex({ createdAt: -1 })  // -1 = по убыванию
```

**2. Compound Index (Составной индекс)**
```javascript
// Индекс по нескольким полям
db.orders.createIndex({ customerId: 1, orderDate: -1 })

// Порядок полей важен! Этот индекс поддерживает запросы:
// - { customerId: ... }
// - { customerId: ..., orderDate: ... }
// НО НЕ поддерживает: { orderDate: ... } (без customerId)
```

**3. Multikey Index (Мультиключевой индекс)**
```javascript
// Автоматически создаётся для полей-массивов
db.articles.createIndex({ tags: 1 })

// Теперь быстро ищем по тегам:
db.articles.find({ tags: "mongodb" })
```

**4. Text Index (Текстовый индекс)**
```javascript
// Для полнотекстового поиска
db.articles.createIndex({ title: "text", content: "text" })

// Использование:
db.articles.find({ $text: { $search: "mongodb performance" } })
```

**5. Geospatial Index (Геопространственный индекс)**
```javascript
// 2dsphere для географических данных
db.places.createIndex({ location: "2dsphere" })

// Поиск ближайших точек:
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [37.6173, 55.7558] },
      $maxDistance: 1000  // метры
    }
  }
})
```

**6. Hashed Index (Хеш-индекс)**
```javascript
// Для шардирования по хешу
db.users.createIndex({ oderId: "hashed" })
```

**7. Partial Index (Частичный индекс)**
```javascript
// Индексирует только документы, соответствующие фильтру
db.orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

**8. TTL Index (Индекс с автоудалением)**
```javascript
// Автоматически удаляет документы через указанное время
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // удаление через 1 час
)
```

### Создание и управление индексами

```javascript
// Просмотр всех индексов коллекции
db.users.getIndexes()

// Создание индекса в фоновом режиме (для production)
db.users.createIndex(
  { email: 1 },
  {
    background: true,  // не блокирует операции
    unique: true,      // уникальные значения
    sparse: true,      // пропускает документы без поля
    name: "idx_email"  // имя индекса
  }
)

// Удаление индекса
db.users.dropIndex("idx_email")
db.users.dropIndex({ email: 1 })

// Удаление всех индексов (кроме _id)
db.users.dropIndexes()
```

### Compound Indexes (Составные индексы)

Правила эффективного использования составных индексов:

**Правило ESR (Equality, Sort, Range):**
```javascript
// Порядок полей в составном индексе:
// 1. Equality (точное совпадение)
// 2. Sort (сортировка)
// 3. Range (диапазон)

// Пример запроса:
db.orders.find({
  status: "shipped",        // Equality
  amount: { $gte: 100 }     // Range
}).sort({ orderDate: -1 })  // Sort

// Оптимальный индекс:
db.orders.createIndex({ status: 1, orderDate: -1, amount: 1 })
```

**Prefix правило:**
```javascript
// Составной индекс { a: 1, b: 1, c: 1 } поддерживает:
// - запросы по { a }
// - запросы по { a, b }
// - запросы по { a, b, c }
// НО НЕ поддерживает:
// - запросы только по { b } или { c }
// - запросы по { b, c } без a
```

### Covered Queries (Покрывающие запросы)

Покрывающий запрос — запрос, который полностью выполняется по индексу без обращения к документам:

```javascript
// Создаём индекс
db.users.createIndex({ email: 1, name: 1 })

// Покрывающий запрос — все поля есть в индексе
db.users.find(
  { email: "user@example.com" },  // фильтр по индексированному полю
  { email: 1, name: 1, _id: 0 }   // projection только индексированных полей
)

// Проверка через explain():
db.users.find(
  { email: "user@example.com" },
  { email: 1, name: 1, _id: 0 }
).explain("executionStats")

// В выводе должно быть:
// "totalDocsExamined": 0  — документы не читались
// "stage": "IXSCAN"       — только сканирование индекса
```

## Профилирование запросов

### Метод explain()

`explain()` — главный инструмент анализа производительности запросов:

```javascript
// Три режима verbosity:
db.collection.find({}).explain("queryPlanner")       // план запроса
db.collection.find({}).explain("executionStats")     // статистика выполнения
db.collection.find({}).explain("allPlansExecution")  // все рассмотренные планы

// Практический пример:
db.orders.find({
  status: "pending",
  amount: { $gte: 1000 }
}).explain("executionStats")
```

**Ключевые метрики explain():**

```javascript
{
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 42,           // возвращено документов
    "executionTimeMillis": 15, // время выполнения (мс)
    "totalKeysExamined": 100,  // просмотрено ключей индекса
    "totalDocsExamined": 42,   // просмотрено документов

    // Идеальное соотношение:
    // totalDocsExamined ≈ nReturned
    // Если totalDocsExamined >> nReturned — неэффективный индекс
  },
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",        // этапы выполнения
      "inputStage": {
        "stage": "IXSCAN",     // сканирование индекса
        "indexName": "status_1_amount_1"
      }
    }
  }
}
```

**Типы stage:**
- `COLLSCAN` — полное сканирование коллекции (плохо!)
- `IXSCAN` — сканирование индекса (хорошо)
- `FETCH` — извлечение документов
- `SORT` — сортировка в памяти (может быть проблемой)
- `PROJECTION` — применение проекции

### Database Profiler

Профайлер записывает медленные операции в специальную коллекцию:

```javascript
// Включение профайлера
// Уровень 0: выключен
// Уровень 1: только медленные запросы (> slowms)
// Уровень 2: все запросы

// Включить для запросов > 100мс
db.setProfilingLevel(1, { slowms: 100 })

// Включить для всех запросов
db.setProfilingLevel(2)

// Проверить текущий уровень
db.getProfilingStatus()

// Анализ профайлера
db.system.profile.find().sort({ ts: -1 }).limit(10)

// Найти самые медленные запросы
db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ millis: -1 })

// Найти запросы с COLLSCAN
db.system.profile.find({
  "planSummary": "COLLSCAN"
})

// Выключить профайлер
db.setProfilingLevel(0)
```

**Пример вывода профайлера:**
```javascript
{
  "op": "query",
  "ns": "mydb.orders",
  "command": {
    "find": "orders",
    "filter": { "status": "pending" }
  },
  "keysExamined": 0,
  "docsExamined": 150000,  // Проблема! Полное сканирование
  "millis": 523,
  "planSummary": "COLLSCAN",
  "ts": ISODate("2024-01-15T10:30:00Z")
}
```

## Оптимизация схемы данных

### Embedding vs Referencing

**Embedding (вложенные документы):**
```javascript
// Хорошо для данных, которые читаются вместе
{
  _id: ObjectId("..."),
  name: "John",
  address: {
    street: "123 Main St",
    city: "Moscow",
    zip: "101000"
  },
  orders: [
    { productId: 1, quantity: 2, price: 100 },
    { productId: 2, quantity: 1, price: 50 }
  ]
}

// Плюсы: одна операция чтения
// Минусы: дублирование, лимит 16MB на документ
```

**Referencing (ссылки):**
```javascript
// Коллекция users
{ _id: ObjectId("user1"), name: "John" }

// Коллекция orders
{
  _id: ObjectId("order1"),
  userId: ObjectId("user1"),  // ссылка
  products: [...]
}

// Плюсы: нормализация, нет дублирования
// Минусы: дополнительные запросы ($lookup)
```

**Рекомендации:**
- Embed: один-к-одному, один-к-немногим, данные читаются вместе
- Reference: один-ко-многим (большое количество), много-ко-многим

### Bucket Pattern (Паттерн корзины)

Эффективен для временных рядов:

```javascript
// Вместо одного документа на каждое измерение
// Группируем измерения в "корзины"
{
  sensorId: "sensor_001",
  date: ISODate("2024-01-15"),
  measurements: [
    { time: "10:00", value: 25.5 },
    { time: "10:01", value: 25.7 },
    { time: "10:02", value: 25.6 }
    // ... до N измерений в корзине
  ],
  count: 3,
  sum: 76.8,
  avg: 25.6
}

// Преимущества:
// - Меньше документов = меньше индексных записей
// - Агрегации уже предвычислены
// - Эффективнее для time-series данных
```

### Computed Pattern (Паттерн вычислений)

Предвычисление частых агрегаций:

```javascript
// Вместо подсчёта при каждом запросе
// Храним вычисленные значения
{
  _id: ObjectId("..."),
  productId: "prod_001",
  reviews: [
    { userId: "u1", rating: 5, text: "..." },
    { userId: "u2", rating: 4, text: "..." }
  ],
  // Предвычисленные метрики
  reviewCount: 2,
  ratingSum: 9,
  avgRating: 4.5
}

// Обновление при добавлении отзыва
db.products.updateOne(
  { _id: productId },
  {
    $push: { reviews: newReview },
    $inc: { reviewCount: 1, ratingSum: newReview.rating },
    $set: { avgRating: { $divide: ["$ratingSum", "$reviewCount"] } }
  }
)
```

### Subset Pattern (Паттерн подмножества)

Хранение часто используемых данных отдельно:

```javascript
// Основной документ с полными данными
// Коллекция product_details
{
  _id: ObjectId("..."),
  name: "Laptop",
  description: "Very long description...",
  specs: { /* много данных */ },
  reviews: [ /* много отзывов */ ]
}

// Легковесный документ для списков
// Коллекция product_summaries
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),  // ссылка на полный документ
  name: "Laptop",
  price: 999,
  thumbnail: "url...",
  avgRating: 4.5
}
```

## Настройка WiredTiger

WiredTiger — движок хранения по умолчанию в MongoDB. Правильная настройка критична для производительности.

### Кэш WiredTiger

```javascript
// Размер кэша WiredTiger (по умолчанию: 50% RAM - 1GB)
// Настройка в mongod.conf:
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  // Установить явно

// Мониторинг использования кэша
db.serverStatus().wiredTiger.cache
{
  "bytes currently in the cache": 1073741824,
  "maximum bytes configured": 4294967296,
  "bytes read into cache": 536870912,
  "bytes written from cache": 268435456,
  "pages evicted": 1000
}
```

### Компрессия

```javascript
// Настройка компрессии в mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy  // snappy (по умолчанию), zlib, zstd, none
    indexConfig:
      prefixCompression: true

// Варианты компрессии:
// - none: без сжатия (максимальная скорость)
// - snappy: баланс скорость/сжатие (по умолчанию)
// - zlib: лучшее сжатие, медленнее
// - zstd: хорошее сжатие, быстрее zlib (рекомендуется)
```

### Настройка журналирования

```javascript
// mongod.conf
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  // интервал записи журнала (мс)

// Для высокой производительности (с риском потери данных):
storage:
  journal:
    commitIntervalMs: 300  # увеличить интервал
```

### Настройка Checkpoint

```javascript
// mongod.conf
storage:
  wiredTiger:
    engineConfig:
      journalCompressor: snappy
      # Checkpoint interval (по умолчанию 60 секунд)
```

## Мониторинг производительности

### mongostat

`mongostat` — утилита для мониторинга в реальном времени:

```bash
# Базовое использование
mongostat

# С указанием интервала (в секундах)
mongostat --rowcount 10 2  # 10 строк, каждые 2 секунды

# Подключение к конкретному серверу
mongostat --host localhost:27017 --username admin --password pass
```

**Вывод mongostat:**
```
insert query update delete getmore command dirty used flushes vsize  res qr|qw ar|aw netIn netOut conn time
  *0    *0     *0     *0       0     2|0  0.0% 0.0%       0  1.5G  85M  0|0   0|0  157b  60.3k    4 10:30:01
```

**Ключевые метрики:**
- `insert/query/update/delete` — операции в секунду
- `dirty` — процент грязных страниц в кэше (> 20% — проблема)
- `used` — использование кэша
- `qr|qw` — очередь чтения|записи (> 0 — перегрузка)
- `ar|aw` — активные операции чтения|записи

### mongotop

`mongotop` — время, потраченное на операции по коллекциям:

```bash
# Базовое использование
mongotop

# С интервалом 5 секунд
mongotop 5

# Вывод
                    ns    total    read    write
    mydb.orders         123ms    100ms     23ms
    mydb.users           45ms     40ms      5ms
    mydb.products        12ms     10ms      2ms
```

### db.serverStatus()

Полная статистика сервера:

```javascript
// Общая статистика
db.serverStatus()

// Отдельные секции
db.serverStatus().connections   // соединения
db.serverStatus().opcounters    // счётчики операций
db.serverStatus().mem           // память
db.serverStatus().wiredTiger    // WiredTiger статистика
db.serverStatus().globalLock    // блокировки

// Пример анализа
const status = db.serverStatus();

// Соотношение чтений/записей
print("Reads:", status.opcounters.query);
print("Writes:", status.opcounters.insert + status.opcounters.update + status.opcounters.delete);

// Активные соединения
print("Current connections:", status.connections.current);
print("Available connections:", status.connections.available);
```

### db.currentOp()

Текущие операции:

```javascript
// Все текущие операции
db.currentOp()

// Только активные операции
db.currentOp({ active: true })

// Операции дольше 3 секунд
db.currentOp({
  active: true,
  secs_running: { $gt: 3 }
})

// Завершить зависшую операцию
db.killOp(opId)
```

### $indexStats

Статистика использования индексов:

```javascript
db.orders.aggregate([{ $indexStats: {} }])

// Вывод
{
  "name": "status_1",
  "accesses": {
    "ops": 15234,      // количество использований
    "since": ISODate("2024-01-01T00:00:00Z")
  }
}

// Найти неиспользуемые индексы
db.orders.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": 0 } }
])
```

## Best Practices

### Индексирование

1. **Создавайте индексы для частых запросов**
```javascript
// Анализируйте паттерны запросов
db.orders.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } }
])

// Создавайте индексы для WHERE и ORDER BY
db.orders.createIndex({ status: 1, createdAt: -1 })
```

2. **Избегайте избыточных индексов**
```javascript
// Плохо: дублирующие индексы
db.users.createIndex({ email: 1 })
db.users.createIndex({ email: 1, name: 1 })  // первый индекс избыточен

// Хорошо: только составной индекс
db.users.createIndex({ email: 1, name: 1 })
```

3. **Используйте partial индексы для разреженных данных**
```javascript
// Индексируем только активные пользователи
db.users.createIndex(
  { lastActivity: -1 },
  { partialFilterExpression: { status: "active" } }
)
```

### Запросы

1. **Ограничивайте возвращаемые поля**
```javascript
// Плохо
db.users.find({ status: "active" })

// Хорошо — только нужные поля
db.users.find(
  { status: "active" },
  { name: 1, email: 1 }
)
```

2. **Используйте limit() и skip() правильно**
```javascript
// Плохо для глубокой пагинации
db.orders.find().skip(100000).limit(10)

// Хорошо — пагинация по курсору
db.orders.find({ _id: { $gt: lastSeenId } }).limit(10)
```

3. **Избегайте $regex без якоря**
```javascript
// Плохо — полное сканирование
db.users.find({ email: { $regex: "gmail" } })

// Хорошо — использует индекс
db.users.find({ email: { $regex: "^john" } })
```

### Схема данных

1. **Ограничивайте размер массивов**
```javascript
// Используйте $slice при обновлении
db.posts.updateOne(
  { _id: postId },
  {
    $push: {
      comments: {
        $each: [newComment],
        $slice: -100  // хранить только последние 100
      }
    }
  }
)
```

2. **Денормализуйте с умом**
```javascript
// Дублируйте только редко изменяемые данные
{
  orderId: ObjectId("..."),
  customer: {
    _id: ObjectId("..."),
    name: "John",      // редко меняется — ок
    // email: "..."    // часто меняется — лучше reference
  }
}
```

### Настройки сервера

1. **Правильный размер кэша**
```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      # Для выделенного сервера: 60-70% RAM
      # Для shared сервера: меньше, чтобы оставить память другим
      cacheSizeGB: 8
```

2. **Connection pooling**
```javascript
// В приложении используйте пул соединений
const client = new MongoClient(uri, {
  maxPoolSize: 100,       // максимум соединений
  minPoolSize: 10,        // минимум соединений
  maxIdleTimeMS: 30000    // время жизни idle соединения
});
```

## Типичные проблемы и их решения

### Проблема 1: Медленные запросы (COLLSCAN)

**Симптомы:**
- Высокое время отклика
- `explain()` показывает COLLSCAN

**Решение:**
```javascript
// 1. Найти проблемные запросы
db.setProfilingLevel(1, { slowms: 100 })

// 2. Проанализировать
db.system.profile.find({ planSummary: "COLLSCAN" })

// 3. Создать подходящий индекс
db.collection.createIndex({ fieldUsedInQuery: 1 })
```

### Проблема 2: Высокое использование памяти

**Симптомы:**
- `dirty` в mongostat > 20%
- Частые evictions в кэше

**Решение:**
```javascript
// 1. Проверить размер рабочего набора
db.stats().dataSize  // должен помещаться в кэш

// 2. Увеличить кэш WiredTiger или добавить RAM

// 3. Оптимизировать запросы — читать меньше данных
db.users.find(
  { status: "active" },
  { name: 1, email: 1 }  // только нужные поля
).limit(100)
```

### Проблема 3: Блокировки и очереди

**Симптомы:**
- `qr|qw > 0` в mongostat
- Высокая задержка

**Решение:**
```javascript
// 1. Найти долгие операции
db.currentOp({ active: true, secs_running: { $gt: 5 } })

// 2. Завершить если нужно
db.killOp(opId)

// 3. Оптимизировать операции записи
// Используйте bulk операции вместо одиночных
const bulk = db.orders.initializeUnorderedBulkOp();
orders.forEach(order => bulk.insert(order));
bulk.execute();
```

### Проблема 4: Неиспользуемые индексы

**Симптомы:**
- Много индексов, медленные записи
- Большой размер на диске

**Решение:**
```javascript
// 1. Найти неиспользуемые индексы
db.orders.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 10 } } }
])

// 2. Удалить ненужные
db.orders.dropIndex("unused_index_name")
```

### Проблема 5: Проблемы с пагинацией

**Симптомы:**
- skip() с большими значениями очень медленный

**Решение:**
```javascript
// Плохо: O(n) сложность
db.orders.find().sort({ createdAt: -1 }).skip(100000).limit(10)

// Хорошо: O(1) с курсором
// Первая страница
const firstPage = await db.orders
  .find()
  .sort({ createdAt: -1 })
  .limit(10)
  .toArray();

// Следующие страницы
const lastId = firstPage[firstPage.length - 1]._id;
const nextPage = await db.orders
  .find({ _id: { $lt: lastId } })
  .sort({ createdAt: -1 })
  .limit(10)
  .toArray();
```

### Проблема 6: Write Concern и производительность

**Симптомы:**
- Медленные записи
- Нужен баланс между надёжностью и скоростью

**Решение:**
```javascript
// Для критичных данных
db.orders.insertOne(order, { writeConcern: { w: "majority", j: true } })

// Для логов и метрик (допустима потеря)
db.logs.insertOne(log, { writeConcern: { w: 0 } })

// Для bulk-операций
db.collection.insertMany(docs, {
  ordered: false,  // продолжать при ошибках
  writeConcern: { w: 1 }
})
```

## Чек-лист оптимизации

- [ ] Все частые запросы используют индексы (нет COLLSCAN)
- [ ] Составные индексы следуют правилу ESR
- [ ] Нет избыточных или неиспользуемых индексов
- [ ] Запросы возвращают только нужные поля (projection)
- [ ] Схема оптимизирована под паттерны чтения
- [ ] WiredTiger cache настроен под нагрузку
- [ ] Настроен мониторинг (mongostat, mongotop, APM)
- [ ] Медленные запросы логируются (profiler)
- [ ] Connection pooling настроен в приложении
- [ ] Регулярный анализ explain() для критичных запросов
