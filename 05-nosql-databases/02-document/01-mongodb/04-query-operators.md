# Операторы запросов MongoDB

## Обзор операторов

MongoDB предоставляет богатый набор операторов для:
- **Query operators** — фильтрация документов
- **Update operators** — модификация документов
- **Aggregation operators** — обработка данных в pipeline

---

## Comparison Operators (Сравнение)

### $eq — равенство

```javascript
// Явное использование
db.users.find({ age: { $eq: 25 } })

// Неявное (эквивалентно)
db.users.find({ age: 25 })

// С вложенными полями
db.users.find({ "address.city": { $eq: "Москва" } })
```

### $ne — не равно

```javascript
// Все, кроме указанного значения
db.users.find({ status: { $ne: "deleted" } })

// Также включает документы без этого поля
db.users.find({ role: { $ne: "admin" } })  // + документы без role
```

### $gt, $gte — больше / больше или равно

```javascript
// Больше
db.products.find({ price: { $gt: 1000 } })

// Больше или равно
db.users.find({ age: { $gte: 18 } })

// С датами
db.orders.find({ created_at: { $gte: ISODate("2024-01-01") } })
```

### $lt, $lte — меньше / меньше или равно

```javascript
// Меньше
db.products.find({ stock: { $lt: 10 } })  // Заканчивающиеся товары

// Меньше или равно
db.users.find({ age: { $lte: 65 } })

// Комбинация — диапазон
db.products.find({ price: { $gte: 100, $lte: 500 } })
```

### $in — входит в список

```javascript
// Любое из значений
db.users.find({ status: { $in: ["active", "pending"] } })

// С массивами — хотя бы один элемент
db.posts.find({ tags: { $in: ["mongodb", "database"] } })

// С ObjectId
db.orders.find({
  user_id: { $in: [ObjectId("..."), ObjectId("...")] }
})
```

### $nin — не входит в список

```javascript
// Ни одно из значений
db.users.find({ role: { $nin: ["admin", "moderator"] } })

// Также включает документы без поля
```

---

## Logical Operators (Логические)

### $and — логическое И

```javascript
// Явное использование
db.products.find({
  $and: [
    { price: { $gte: 100 } },
    { price: { $lte: 500 } },
    { category: "electronics" }
  ]
})

// Неявное (эквивалентно, когда поля разные)
db.products.find({
  price: { $gte: 100, $lte: 500 },
  category: "electronics"
})

// $and необходим для условий на одно поле
db.products.find({
  $and: [
    { tags: "sale" },
    { tags: "new" }
  ]
})
```

### $or — логическое ИЛИ

```javascript
// Любое из условий
db.users.find({
  $or: [
    { email: "admin@example.com" },
    { role: "admin" }
  ]
})

// Комбинация с другими условиями
db.orders.find({
  status: "active",
  $or: [
    { priority: "high" },
    { amount: { $gte: 10000 } }
  ]
})
```

### $not — логическое НЕ

```javascript
// Инвертирует условие
db.products.find({
  price: { $not: { $gt: 1000 } }  // price <= 1000 или нет поля
})

// С регулярным выражением
db.users.find({
  name: { $not: /^admin/i }
})
```

### $nor — НИ ОДНО из условий

```javascript
// Ни одно условие не выполняется
db.users.find({
  $nor: [
    { status: "deleted" },
    { status: "banned" },
    { age: { $lt: 18 } }
  ]
})
```

---

## Element Operators (Элементы)

### $exists — наличие поля

```javascript
// Поле существует
db.users.find({ phone: { $exists: true } })

// Поле не существует
db.users.find({ deleted_at: { $exists: false } })

// Комбинация — поле есть и не null
db.users.find({
  phone: { $exists: true, $ne: null }
})
```

### $type — проверка типа

```javascript
// По имени типа
db.users.find({ age: { $type: "int" } })
db.users.find({ age: { $type: "number" } })  // int, long, double, decimal

// По номеру BSON типа
db.users.find({ age: { $type: 16 } })  // int32

// Несколько типов
db.data.find({
  value: { $type: ["string", "int"] }
})

// Распространённые типы:
// "double" (1), "string" (2), "object" (3), "array" (4)
// "binData" (5), "objectId" (7), "bool" (8), "date" (9)
// "null" (10), "regex" (11), "int" (16), "long" (18)
// "decimal" (19), "number" (any numeric)
```

---

## Evaluation Operators (Вычисление)

### $regex — регулярные выражения

```javascript
// Базовое использование
db.users.find({ email: { $regex: /@gmail\.com$/ } })

// С опциями
db.users.find({
  name: { $regex: "иван", $options: "i" }  // case-insensitive
})

// Сокращённая форма
db.users.find({ name: /^Иван/i })

// Опции:
// i — case-insensitive
// m — multiline (^ и $ для каждой строки)
// s — dot matches newline
// x — extended (игнорировать пробелы)
```

### $expr — выражения агрегации

```javascript
// Сравнение полей между собой
db.orders.find({
  $expr: { $gt: ["$amount", "$discount"] }
})

// Сложные вычисления
db.products.find({
  $expr: {
    $lt: [
      { $multiply: ["$price", "$quantity"] },
      1000
    ]
  }
})

// С условной логикой
db.orders.find({
  $expr: {
    $and: [
      { $eq: ["$status", "pending"] },
      { $gt: [{ $subtract: [new Date(), "$created_at"] }, 86400000] }
    ]
  }
})
```

### $mod — остаток от деления

```javascript
// Документы где field % divisor == remainder
db.items.find({ quantity: { $mod: [4, 0] } })  // Кратные 4

// Чётные числа
db.items.find({ count: { $mod: [2, 0] } })

// Нечётные числа
db.items.find({ count: { $mod: [2, 1] } })
```

### $text — полнотекстовый поиск

```javascript
// Требуется текстовый индекс
db.articles.createIndex({ title: "text", content: "text" })

// Поиск
db.articles.find({ $text: { $search: "mongodb database" } })

// Точная фраза
db.articles.find({ $text: { $search: "\"NoSQL database\"" } })

// Исключение слов
db.articles.find({ $text: { $search: "mongodb -sql" } })

// С языком
db.articles.find({
  $text: { $search: "база данных", $language: "russian" }
})

// Сортировка по релевантности
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

### $where — JavaScript выражение

```javascript
// Выполняет JavaScript (медленно, избегайте)
db.users.find({
  $where: function() {
    return this.name.length > 10
  }
})

// Строковая форма
db.users.find({
  $where: "this.password.length >= 8"
})

// Предпочитайте $expr или агрегацию!
```

### $jsonSchema — валидация схемы

```javascript
// Поиск документов, соответствующих схеме
db.users.find({
  $jsonSchema: {
    required: ["email", "name"],
    properties: {
      email: { bsonType: "string" },
      age: { bsonType: "int", minimum: 18 }
    }
  }
})
```

---

## Array Operators (Массивы)

### $all — все элементы присутствуют

```javascript
// Массив содержит все указанные элементы
db.posts.find({ tags: { $all: ["mongodb", "tutorial"] } })

// С $elemMatch внутри
db.inventory.find({
  items: {
    $all: [
      { $elemMatch: { size: "M", color: "blue" } },
      { $elemMatch: { size: "L", color: "red" } }
    ]
  }
})
```

### $elemMatch — элемент соответствует условиям

```javascript
// Один элемент массива соответствует всем условиям
db.orders.find({
  items: {
    $elemMatch: {
      product: "Laptop",
      quantity: { $gte: 2 },
      price: { $lt: 100000 }
    }
  }
})

// Разница с обычным запросом:
// { "items.product": "Laptop", "items.quantity": 2 }
// — условия могут выполняться разными элементами

// { items: { $elemMatch: { product: "Laptop", quantity: 2 } } }
// — все условия для ОДНОГО элемента
```

### $size — точный размер массива

```javascript
// Массив с точным количеством элементов
db.posts.find({ tags: { $size: 3 } })

// Для диапазона используйте $expr:
db.posts.find({
  $expr: { $gte: [{ $size: "$tags" }, 3] }
})

// Или $where (медленнее):
db.posts.find({ $where: "this.tags.length >= 3" })
```

### Индексация элементов массива

```javascript
// По индексу
db.users.find({ "scores.0": { $gte: 90 } })  // Первый элемент >= 90

// Последний элемент (через $expr)
db.users.find({
  $expr: {
    $gte: [{ $arrayElemAt: ["$scores", -1] }, 90]
  }
})
```

---

## Update Operators (Обновление)

### Field Update Operators

#### $set — установить значение

```javascript
db.users.updateOne(
  { _id: ObjectId("...") },
  {
    $set: {
      name: "Новое имя",
      "address.city": "Москва",
      updatedAt: new Date()
    }
  }
)
```

#### $unset — удалить поле

```javascript
db.users.updateOne(
  { _id: ObjectId("...") },
  { $unset: { temporaryField: "", oldField: "" } }  // Значение игнорируется
)
```

#### $inc — инкремент

```javascript
// Увеличить
db.products.updateOne(
  { _id: ObjectId("...") },
  { $inc: { views: 1, stock: -1 } }
)

// Работает с int, long, double, decimal
```

#### $mul — умножение

```javascript
db.products.updateMany(
  { category: "sale" },
  { $mul: { price: 0.9 } }  // Скидка 10%
)
```

#### $min, $max — обновить если меньше/больше

```javascript
// Обновить только если новое значение меньше
db.stats.updateOne(
  { _id: "record" },
  { $min: { lowScore: 50 } }
)

// Обновить только если новое значение больше
db.stats.updateOne(
  { _id: "record" },
  { $max: { highScore: 100 } }
)
```

#### $rename — переименовать поле

```javascript
db.users.updateMany(
  {},
  { $rename: { "phone": "mobile", "addr": "address" } }
)
```

#### $setOnInsert — установить только при upsert

```javascript
db.users.updateOne(
  { email: "new@example.com" },
  {
    $set: { lastLogin: new Date() },
    $setOnInsert: { createdAt: new Date(), role: "user" }
  },
  { upsert: true }
)
```

#### $currentDate — текущая дата

```javascript
db.orders.updateOne(
  { _id: ObjectId("...") },
  {
    $set: { status: "completed" },
    $currentDate: {
      updatedAt: true,                    // Date
      "history.timestamp": { $type: "timestamp" }  // Timestamp
    }
  }
)
```

---

### Array Update Operators

#### $push — добавить в конец массива

```javascript
// Один элемент
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $push: { tags: "new-tag" } }
)

// Несколько элементов
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $push: { tags: { $each: ["tag1", "tag2", "tag3"] } } }
)

// С модификаторами
db.posts.updateOne(
  { _id: ObjectId("...") },
  {
    $push: {
      comments: {
        $each: [{ user: "Иван", text: "Комментарий" }],
        $position: 0,        // В начало
        $slice: -10,         // Оставить последние 10
        $sort: { date: -1 }  // Отсортировать
      }
    }
  }
)
```

#### $addToSet — добавить уникальный

```javascript
// Добавить если не существует
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $addToSet: { tags: "mongodb" } }
)

// Несколько значений
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $addToSet: { tags: { $each: ["tag1", "tag2"] } } }
)
```

#### $pop — удалить первый/последний

```javascript
// Удалить последний
db.lists.updateOne(
  { _id: ObjectId("...") },
  { $pop: { items: 1 } }
)

// Удалить первый
db.lists.updateOne(
  { _id: ObjectId("...") },
  { $pop: { items: -1 } }
)
```

#### $pull — удалить по условию

```javascript
// Удалить конкретное значение
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $pull: { tags: "old-tag" } }
)

// Удалить по условию
db.orders.updateOne(
  { _id: ObjectId("...") },
  { $pull: { items: { quantity: { $lte: 0 } } } }
)
```

#### $pullAll — удалить несколько значений

```javascript
db.posts.updateOne(
  { _id: ObjectId("...") },
  { $pullAll: { tags: ["tag1", "tag2", "tag3"] } }
)
```

#### $ (positional) — обновить найденный элемент

```javascript
// Обновить первый совпавший элемент
db.orders.updateOne(
  { _id: ObjectId("..."), "items.product": "Laptop" },
  { $set: { "items.$.price": 90000 } }
)
```

#### $[] — обновить все элементы

```javascript
// Обновить все элементы массива
db.orders.updateOne(
  { _id: ObjectId("...") },
  { $inc: { "items.$[].price": 100 } }
)
```

#### $[\<identifier\>] — обновить по фильтру

```javascript
// Обновить элементы по условию
db.orders.updateOne(
  { _id: ObjectId("...") },
  { $set: { "items.$[elem].discount": true } },
  { arrayFilters: [{ "elem.price": { $gte: 1000 } }] }
)

// Несколько фильтров
db.orders.updateOne(
  { _id: ObjectId("...") },
  {
    $set: {
      "items.$[expensive].discount": 0.1,
      "items.$[cheap].discount": 0.05
    }
  },
  {
    arrayFilters: [
      { "expensive.price": { $gte: 1000 } },
      { "cheap.price": { $lt: 1000 } }
    ]
  }
)
```

---

## Bitwise Operators

```javascript
// $bitsAllSet — все биты установлены
db.items.find({ flags: { $bitsAllSet: [1, 5] } })  // биты 1 и 5

// $bitsAnySet — любой бит установлен
db.items.find({ flags: { $bitsAnySet: 35 } })  // маска

// $bitsAllClear — все биты сброшены
db.items.find({ flags: { $bitsAllClear: [1, 5] } })

// $bitsAnyClear — любой бит сброшен
db.items.find({ flags: { $bitsAnyClear: 35 } })

// Update с $bit
db.items.updateOne(
  { _id: ObjectId("...") },
  { $bit: { flags: { or: 4, and: 7, xor: 2 } } }
)
```

---

## Geospatial Operators

### Query Operators

```javascript
// Создание геоиндекса
db.places.createIndex({ location: "2dsphere" })

// $near — ближайшие точки
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [37.6173, 55.7558] },
      $maxDistance: 5000,  // метры
      $minDistance: 100
    }
  }
})

// $geoWithin — внутри области
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [37.5, 55.7], [37.7, 55.7],
          [37.7, 55.8], [37.5, 55.8],
          [37.5, 55.7]
        ]]
      }
    }
  }
})

// $geoIntersects — пересекает область
db.routes.find({
  path: {
    $geoIntersects: {
      $geometry: { type: "Point", coordinates: [37.6, 55.75] }
    }
  }
})
```

---

## Projection Operators

### $, $elemMatch, $slice в проекции

```javascript
// $ — первый совпавший элемент массива
db.posts.find(
  { "comments.author": "Иван" },
  { "comments.$": 1 }
)

// $elemMatch — элемент по условию
db.posts.find(
  { _id: ObjectId("...") },
  {
    comments: {
      $elemMatch: { rating: { $gte: 4 } }
    }
  }
)

// $slice — часть массива
db.posts.find(
  {},
  { comments: { $slice: 5 } }       // первые 5
)

db.posts.find(
  {},
  { comments: { $slice: -3 } }      // последние 3
)

db.posts.find(
  {},
  { comments: { $slice: [10, 5] } }  // skip 10, limit 5
)

// $meta — метаданные
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
)
```

---

## Практические примеры

### Сложный поиск

```javascript
// Найти активных пользователей из Москвы или СПб,
// возрастом от 25 до 40, с подтверждённым email
db.users.find({
  $and: [
    { active: true },
    { emailVerified: true },
    { age: { $gte: 25, $lte: 40 } },
    { $or: [
      { "address.city": "Москва" },
      { "address.city": "Санкт-Петербург" }
    ]}
  ]
})
```

### Сложное обновление

```javascript
// Добавить тег, обновить счётчик, записать дату
db.posts.updateOne(
  { _id: ObjectId("..."), tags: { $ne: "featured" } },
  {
    $addToSet: { tags: "featured" },
    $inc: { featureCount: 1 },
    $currentDate: { lastFeatured: true },
    $push: {
      history: {
        $each: [{ action: "featured", date: new Date() }],
        $slice: -100
      }
    }
  }
)
```

### Обновление вложенного массива

```javascript
// Увеличить цену товара в заказе
db.orders.updateOne(
  {
    _id: ObjectId("..."),
    "items.productId": ObjectId("product123")
  },
  {
    $mul: { "items.$.price": 1.1 },
    $set: { "items.$.updatedAt": new Date() }
  }
)
```

---

## Best Practices

1. **Используйте индексы** — операторы $regex, $where, $expr без индексов медленные
2. **Избегайте $where** — предпочитайте $expr для сложных условий
3. **$elemMatch для массивов** — когда нужно несколько условий для одного элемента
4. **Комбинируйте операторы** — один updateOne с несколькими операторами эффективнее
5. **arrayFilters для сложных обновлений** — точечное обновление элементов массива
6. **Проверяйте план запроса** — explain() покажет использование индексов
7. **$in вместо $or** — для одного поля $in эффективнее
