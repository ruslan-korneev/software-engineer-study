# Коллекции и методы MongoDB

## Что такое коллекция

**Коллекция (Collection)** — это группа документов MongoDB, аналог таблицы в реляционных базах данных. В отличие от таблиц, коллекции не требуют заранее определённой схемы.

### Характеристики коллекций

- **Гибкая схема**: документы в одной коллекции могут иметь разные поля
- **Динамическое создание**: коллекция создаётся автоматически при первой вставке
- **Именование**: имена регистрозависимы, максимум 120 байт
- **Namespace**: полное имя = `database.collection` (ограничение 120 байт)

---

## Создание и управление коллекциями

### Неявное создание

```javascript
// Коллекция создаётся автоматически при первой вставке
db.users.insertOne({ name: "Иван" })

// Проверка существования
db.getCollectionNames()
// или
show collections
```

### Явное создание

```javascript
// Базовое создание
db.createCollection("logs")

// С опциями
db.createCollection("logs", {
  capped: true,           // Ограниченная коллекция
  size: 10485760,         // Максимум 10MB
  max: 5000,              // Максимум 5000 документов
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
})

// С валидацией схемы
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        name: {
          bsonType: "string",
          minLength: 2
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        }
      }
    }
  },
  validationLevel: "strict",    // strict | moderate
  validationAction: "error"     // error | warn
})
```

### Управление коллекциями

```javascript
// Переименование
db.oldName.renameCollection("newName")

// Удаление коллекции
db.users.drop()

// Получение информации
db.users.stats()
db.getCollectionInfos({ name: "users" })

// Изменение опций
db.runCommand({
  collMod: "users",
  validator: { $jsonSchema: { ... } },
  validationLevel: "moderate"
})
```

---

## CRUD-операции

### Create (Вставка)

#### insertOne

```javascript
// Вставка одного документа
const result = db.users.insertOne({
  name: "Алексей Петров",
  email: "alex@example.com",
  age: 28,
  created_at: new Date()
})

// Результат
{
  acknowledged: true,
  insertedId: ObjectId("...")
}
```

#### insertMany

```javascript
// Вставка нескольких документов
const result = db.users.insertMany([
  { name: "Мария", age: 25 },
  { name: "Дмитрий", age: 32 },
  { name: "Ольга", age: 29 }
])

// С опциями
db.users.insertMany(
  [
    { _id: 1, name: "Один" },
    { _id: 1, name: "Дубликат" },  // Ошибка дубликата
    { _id: 2, name: "Два" }
  ],
  { ordered: false }  // Продолжить вставку после ошибки
)

// Результат
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId("..."),
    '1': ObjectId("..."),
    '2': ObjectId("...")
  }
}
```

---

### Read (Чтение)

#### find

```javascript
// Все документы
db.users.find()

// С фильтром
db.users.find({ age: { $gte: 25 } })

// С проекцией
db.users.find(
  { age: { $gte: 25 } },
  { name: 1, email: 1, _id: 0 }  // Включить name и email, исключить _id
)

// Курсор и итерация
const cursor = db.users.find({ active: true })
while (cursor.hasNext()) {
  printjson(cursor.next())
}

// Преобразование в массив
const users = db.users.find().toArray()
```

#### findOne

```javascript
// Первый документ по фильтру
const user = db.users.findOne({ email: "alex@example.com" })

// С проекцией
const user = db.users.findOne(
  { email: "alex@example.com" },
  { name: 1, age: 1 }
)
```

#### Модификаторы курсора

```javascript
// Сортировка
db.users.find().sort({ age: -1, name: 1 })  // age DESC, name ASC

// Ограничение количества
db.users.find().limit(10)

// Пропуск документов
db.users.find().skip(20)

// Пагинация
db.users.find()
  .sort({ created_at: -1 })
  .skip(20)     // Пропустить первые 20
  .limit(10)    // Взять следующие 10

// Подсчёт
db.users.countDocuments({ active: true })
db.users.estimatedDocumentCount()  // Быстрый, но приблизительный

// Distinct значения
db.users.distinct("city")
db.users.distinct("city", { age: { $gte: 18 } })
```

---

### Update (Обновление)

#### updateOne

```javascript
// Обновление одного документа
db.users.updateOne(
  { email: "alex@example.com" },  // Фильтр
  { $set: { age: 29 } }           // Обновление
)

// С опциями
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "Новый пользователь" } },
  { upsert: true }  // Создать, если не существует
)

// Результат
{
  acknowledged: true,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedId: null  // или ObjectId при upsert
}
```

#### updateMany

```javascript
// Обновление всех подходящих документов
db.users.updateMany(
  { age: { $lt: 18 } },
  { $set: { status: "minor" } }
)

// Инкремент всем
db.products.updateMany(
  {},
  { $mul: { price: 1.1 } }  // Увеличить цену на 10%
)
```

#### replaceOne

```javascript
// Полная замена документа (кроме _id)
db.users.replaceOne(
  { email: "alex@example.com" },
  {
    name: "Александр Петров",
    email: "alex@example.com",
    phone: "+7 999 123-45-67",
    updated_at: new Date()
  }
)
```

#### findOneAndUpdate

```javascript
// Найти, обновить и вернуть документ
const user = db.users.findOneAndUpdate(
  { email: "alex@example.com" },
  { $inc: { login_count: 1 }, $set: { last_login: new Date() } },
  {
    returnDocument: "after",  // "before" | "after"
    projection: { password: 0 },
    upsert: false
  }
)
```

---

### Delete (Удаление)

#### deleteOne

```javascript
// Удаление одного документа
db.users.deleteOne({ email: "alex@example.com" })

// Результат
{
  acknowledged: true,
  deletedCount: 1
}
```

#### deleteMany

```javascript
// Удаление всех подходящих документов
db.logs.deleteMany({ created_at: { $lt: ISODate("2023-01-01") } })

// Удаление всех документов (оставляет коллекцию и индексы)
db.logs.deleteMany({})
```

#### findOneAndDelete

```javascript
// Найти, удалить и вернуть документ
const deletedUser = db.users.findOneAndDelete(
  { status: "deleted" },
  { sort: { created_at: 1 } }  // Удалить самый старый
)
```

---

## Bulk Operations

Bulk-операции позволяют выполнять множество операций одним запросом к серверу.

### Ordered Bulk

```javascript
// Операции выполняются последовательно
const bulk = db.users.initializeOrderedBulkOp()

bulk.insert({ name: "Пользователь 1" })
bulk.insert({ name: "Пользователь 2" })
bulk.find({ name: "Старый" }).updateOne({ $set: { name: "Новый" } })
bulk.find({ status: "deleted" }).delete()

const result = bulk.execute()
```

### Unordered Bulk

```javascript
// Операции выполняются параллельно (быстрее)
const bulk = db.users.initializeUnorderedBulkOp()

bulk.insert({ name: "User 1" })
bulk.insert({ name: "User 2" })
bulk.find({ age: { $lt: 0 } }).delete()

const result = bulk.execute()
```

### bulkWrite

```javascript
// Современный API
db.users.bulkWrite([
  {
    insertOne: {
      document: { name: "Новый", age: 25 }
    }
  },
  {
    updateOne: {
      filter: { name: "Иван" },
      update: { $set: { age: 30 } }
    }
  },
  {
    updateMany: {
      filter: { active: false },
      update: { $set: { status: "inactive" } }
    }
  },
  {
    deleteOne: {
      filter: { status: "deleted" }
    }
  },
  {
    replaceOne: {
      filter: { name: "Старый" },
      replacement: { name: "Новый", updated: true }
    }
  }
], {
  ordered: false,
  writeConcern: { w: "majority" }
})
```

---

## Capped Collections

**Capped Collection** — коллекция фиксированного размера, работающая по принципу кольцевого буфера.

### Создание

```javascript
db.createCollection("logs", {
  capped: true,
  size: 1048576,     // 1 MB максимум
  max: 1000          // 1000 документов максимум
})
```

### Характеристики

- Документы хранятся в порядке вставки
- При достижении лимита старые документы автоматически удаляются
- Нельзя удалять отдельные документы
- Нельзя увеличивать размер документа
- Очень быстрая запись

### Использование

```javascript
// Вставка (обычная)
db.logs.insertOne({ message: "Event occurred", timestamp: new Date() })

// Tailable cursor — ожидание новых документов
const cursor = db.logs.find().tailable().awaitData()
while (cursor.hasNext()) {
  console.log(cursor.next())
}
```

### Когда использовать

- Логирование
- Кеширование последних данных
- Очереди сообщений (простые)
- Мониторинг в реальном времени

---

## Schema Validation

### JSON Schema Validation

```javascript
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      title: "Product validation",
      required: ["name", "price", "category"],
      properties: {
        name: {
          bsonType: "string",
          minLength: 1,
          maxLength: 200,
          description: "Название продукта"
        },
        price: {
          bsonType: "decimal",
          minimum: 0,
          description: "Цена должна быть положительной"
        },
        category: {
          enum: ["electronics", "clothing", "food", "other"],
          description: "Категория из списка"
        },
        tags: {
          bsonType: "array",
          items: {
            bsonType: "string"
          },
          uniqueItems: true
        },
        specs: {
          bsonType: "object",
          additionalProperties: true
        }
      }
    }
  }
})
```

### Уровни валидации

```javascript
// strict — проверять все операции записи (по умолчанию)
// moderate — проверять только insert и update существующих валидных документов

db.runCommand({
  collMod: "products",
  validationLevel: "moderate"
})
```

### Действия при ошибке

```javascript
// error — отклонить операцию (по умолчанию)
// warn — разрешить, но записать в лог

db.runCommand({
  collMod: "products",
  validationAction: "warn"
})
```

### Добавление валидации к существующей коллекции

```javascript
db.runCommand({
  collMod: "existingCollection",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name"],
      properties: {
        name: { bsonType: "string" }
      }
    }
  },
  validationLevel: "moderate"  // Не проверять старые документы
})
```

---

## Методы работы с индексами

```javascript
// Создание индекса
db.users.createIndex({ email: 1 })  // Ascending
db.users.createIndex({ age: -1 })   // Descending

// Составной индекс
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Уникальный индекс
db.users.createIndex({ email: 1 }, { unique: true })

// Sparse индекс (только документы с полем)
db.users.createIndex({ phone: 1 }, { sparse: true })

// TTL индекс (автоудаление)
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }  // Удалять через час
)

// Текстовый индекс
db.articles.createIndex({ title: "text", content: "text" })

// Получение списка индексов
db.users.getIndexes()

// Удаление индекса
db.users.dropIndex("email_1")
db.users.dropIndex({ email: 1 })

// Удаление всех индексов (кроме _id)
db.users.dropIndexes()

// Перестроение индексов
db.users.reIndex()
```

---

## Write Concern

Write Concern определяет уровень подтверждения записи.

```javascript
// На уровне операции
db.users.insertOne(
  { name: "Иван" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)

// Уровни w:
// 0 — без подтверждения (fire-and-forget)
// 1 — подтверждение от primary (по умолчанию)
// "majority" — подтверждение от большинства узлов
// n — подтверждение от n узлов

// j: true — запись в journal (на диск)
// wtimeout — таймаут в миллисекундах
```

---

## Read Concern

Read Concern определяет уровень консистентности чтения.

```javascript
// На уровне операции
db.users.find().readConcern("majority")

// Уровни:
// "local" — последние данные на текущем узле (по умолчанию)
// "available" — как local, но без гарантий для шардированных
// "majority" — данные, подтверждённые большинством
// "linearizable" — отражает все успешные записи до начала чтения
// "snapshot" — для транзакций
```

---

## Полезные методы коллекций

```javascript
// Статистика коллекции
db.users.stats()
db.users.dataSize()
db.users.storageSize()
db.users.totalIndexSize()

// Количество документов
db.users.countDocuments({})
db.users.estimatedDocumentCount()

// Distinct значения
db.users.distinct("city")

// Объяснение плана запроса
db.users.find({ age: 25 }).explain("executionStats")

// Агрегация
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customer_id", total: { $sum: "$amount" } } }
])

// MapReduce (устаревает, используйте aggregate)
db.orders.mapReduce(
  function() { emit(this.customer_id, this.amount) },
  function(key, values) { return Array.sum(values) },
  { out: "customer_totals" }
)

// Watch (Change Streams)
const changeStream = db.users.watch()
changeStream.on("change", (change) => {
  console.log(change)
})
```

---

## Транзакции

```javascript
const session = db.getMongo().startSession()
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
})

try {
  const users = session.getDatabase("mydb").users
  const orders = session.getDatabase("mydb").orders

  users.updateOne(
    { _id: userId },
    { $inc: { balance: -100 } }
  )

  orders.insertOne({
    user_id: userId,
    amount: 100,
    status: "pending"
  })

  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

---

## Best Practices

1. **Используйте bulk-операции** для массовых изменений
2. **Добавляйте валидацию** для важных коллекций
3. **Используйте проекцию** для уменьшения объёма данных
4. **Создавайте индексы** для частых запросов
5. **Используйте Write Concern "majority"** для критичных данных
6. **Мониторьте размер коллекций** и документов
7. **Используйте Capped Collections** для логов и временных данных
8. **Проверяйте план запроса** с помощью explain()
