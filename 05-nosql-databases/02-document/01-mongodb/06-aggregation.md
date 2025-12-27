# Aggregation Framework

## Введение

**Aggregation Framework** — это мощный инструмент MongoDB для обработки и анализа данных. Он позволяет выполнять сложные операции трансформации, группировки, фильтрации и вычислений над коллекциями документов.

В отличие от простых запросов `find()`, aggregation позволяет:
- Группировать документы по полям
- Вычислять агрегированные значения (суммы, средние и т.д.)
- Трансформировать структуру документов
- Объединять данные из нескольких коллекций (аналог JOIN)
- Создавать сложные аналитические отчёты

---

## Концепция Aggregation Pipeline

### Что такое Pipeline?

**Aggregation Pipeline** — это последовательность стадий (stages), через которые проходят документы. Каждая стадия трансформирует входные документы и передаёт результат на следующую стадию.

```
Коллекция → Stage 1 → Stage 2 → Stage 3 → ... → Результат
```

### Базовый синтаксис

```javascript
db.collection.aggregate([
  { $stage1: { /* параметры */ } },
  { $stage2: { /* параметры */ } },
  { $stage3: { /* параметры */ } }
])
```

### Простой пример

```javascript
// Коллекция orders
{ _id: 1, customer: "Alice", amount: 100, status: "completed" }
{ _id: 2, customer: "Bob", amount: 200, status: "completed" }
{ _id: 3, customer: "Alice", amount: 150, status: "pending" }
{ _id: 4, customer: "Charlie", amount: 300, status: "completed" }

// Найти общую сумму завершённых заказов каждого клиента
db.orders.aggregate([
  { $match: { status: "completed" } },           // Фильтруем
  { $group: { _id: "$customer", total: { $sum: "$amount" } } },  // Группируем
  { $sort: { total: -1 } }                       // Сортируем по убыванию
])

// Результат:
{ _id: "Charlie", total: 300 }
{ _id: "Bob", total: 200 }
{ _id: "Alice", total: 100 }
```

---

## Основные стадии Pipeline

### $match — Фильтрация документов

`$match` фильтрует документы, пропуская только те, которые соответствуют условиям. Аналогичен методу `find()`.

```javascript
// Фильтрация по одному условию
db.products.aggregate([
  { $match: { category: "electronics" } }
])

// Множественные условия
db.products.aggregate([
  { $match: {
    category: "electronics",
    price: { $gte: 100, $lte: 500 },
    inStock: true
  }}
])

// Использование $or
db.products.aggregate([
  { $match: {
    $or: [
      { category: "electronics" },
      { category: "computers" }
    ]
  }}
])

// Поиск по массиву
db.products.aggregate([
  { $match: { tags: { $in: ["sale", "new"] } } }
])
```

> **Best Practice**: Размещайте `$match` как можно раньше в pipeline для уменьшения количества обрабатываемых документов.

---

### $group — Группировка документов

`$group` группирует документы по указанному ключу и позволяет применять аккумулирующие операторы.

```javascript
// Базовая группировка
db.sales.aggregate([
  { $group: {
    _id: "$category",              // Поле группировки
    totalSales: { $sum: "$amount" },
    avgSale: { $avg: "$amount" },
    count: { $sum: 1 }
  }}
])

// Группировка по нескольким полям
db.sales.aggregate([
  { $group: {
    _id: {
      year: { $year: "$date" },
      month: { $month: "$date" },
      category: "$category"
    },
    revenue: { $sum: "$amount" }
  }}
])

// Группировка всех документов (без _id)
db.orders.aggregate([
  { $group: {
    _id: null,                     // null = все документы в одну группу
    totalOrders: { $sum: 1 },
    totalRevenue: { $sum: "$amount" },
    avgOrderValue: { $avg: "$amount" }
  }}
])
```

---

### $sort — Сортировка

`$sort` упорядочивает документы по указанным полям.

```javascript
// Сортировка по возрастанию (1) и убыванию (-1)
db.products.aggregate([
  { $sort: { price: 1 } }           // По возрастанию цены
])

db.products.aggregate([
  { $sort: { price: -1 } }          // По убыванию цены
])

// Сортировка по нескольким полям
db.products.aggregate([
  { $sort: { category: 1, price: -1 } }  // Сначала по категории, затем по цене
])

// После группировки
db.sales.aggregate([
  { $group: { _id: "$product", total: { $sum: "$quantity" } } },
  { $sort: { total: -1 } }          // Топ продуктов по продажам
])
```

---

### $project — Проекция и трансформация

`$project` изменяет структуру документов: включает/исключает поля, создаёт новые, переименовывает существующие.

```javascript
// Включение/исключение полей
db.users.aggregate([
  { $project: {
    name: 1,                        // Включить
    email: 1,                       // Включить
    _id: 0,                         // Исключить
    password: 0                     // Исключить
  }}
])

// Переименование полей
db.users.aggregate([
  { $project: {
    userName: "$name",              // Переименовать name → userName
    userEmail: "$email"
  }}
])

// Создание вычисляемых полей
db.products.aggregate([
  { $project: {
    name: 1,
    originalPrice: "$price",
    discountedPrice: { $multiply: ["$price", 0.9] },  // Цена со скидкой 10%
    inStock: { $gt: ["$quantity", 0] }                // Boolean: есть ли в наличии
  }}
])

// Работа со строками
db.users.aggregate([
  { $project: {
    fullName: { $concat: ["$firstName", " ", "$lastName"] },
    emailDomain: { $arrayElemAt: [{ $split: ["$email", "@"] }, 1] },
    nameLength: { $strLenCP: "$firstName" }
  }}
])

// Условная логика
db.products.aggregate([
  { $project: {
    name: 1,
    priceCategory: {
      $switch: {
        branches: [
          { case: { $lt: ["$price", 50] }, then: "budget" },
          { case: { $lt: ["$price", 200] }, then: "mid-range" }
        ],
        default: "premium"
      }
    }
  }}
])

// $cond — тернарный оператор
db.orders.aggregate([
  { $project: {
    orderId: 1,
    status: 1,
    statusLabel: {
      $cond: {
        if: { $eq: ["$status", "completed"] },
        then: "Завершён",
        else: "В обработке"
      }
    }
  }}
])
```

---

### $unwind — Развёртывание массивов

`$unwind` "разворачивает" массив, создавая отдельный документ для каждого элемента массива.

```javascript
// Исходный документ
{ _id: 1, name: "Laptop", tags: ["electronics", "computers", "sale"] }

// После $unwind
db.products.aggregate([
  { $unwind: "$tags" }
])

// Результат:
{ _id: 1, name: "Laptop", tags: "electronics" }
{ _id: 1, name: "Laptop", tags: "computers" }
{ _id: 1, name: "Laptop", tags: "sale" }
```

```javascript
// Практический пример: подсчёт популярности тегов
db.products.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Сохранение документов без массива (preserveNullAndEmptyArrays)
db.products.aggregate([
  { $unwind: {
    path: "$tags",
    preserveNullAndEmptyArrays: true  // Не исключать документы без tags
  }}
])

// Добавление индекса элемента массива
db.products.aggregate([
  { $unwind: {
    path: "$items",
    includeArrayIndex: "itemIndex"    // Добавит поле itemIndex
  }}
])
```

---

### $lookup — JOIN операции

`$lookup` объединяет документы из разных коллекций (аналог SQL LEFT JOIN).

```javascript
// Коллекция orders
{ _id: 1, productId: 101, quantity: 2, customerId: "c1" }
{ _id: 2, productId: 102, quantity: 1, customerId: "c2" }

// Коллекция products
{ _id: 101, name: "Laptop", price: 1000 }
{ _id: 102, name: "Mouse", price: 25 }

// Простой $lookup
db.orders.aggregate([
  { $lookup: {
    from: "products",           // Коллекция для объединения
    localField: "productId",    // Поле в текущей коллекции
    foreignField: "_id",        // Поле в products
    as: "product"               // Имя нового поля (массив)
  }}
])

// Результат:
{
  _id: 1,
  productId: 101,
  quantity: 2,
  customerId: "c1",
  product: [{ _id: 101, name: "Laptop", price: 1000 }]  // Массив!
}
```

```javascript
// Развёртывание результата lookup (получить объект вместо массива)
db.orders.aggregate([
  { $lookup: {
    from: "products",
    localField: "productId",
    foreignField: "_id",
    as: "product"
  }},
  { $unwind: "$product" },      // Развернуть массив в объект
  { $project: {
    _id: 1,
    quantity: 1,
    productName: "$product.name",
    productPrice: "$product.price",
    totalPrice: { $multiply: ["$quantity", "$product.price"] }
  }}
])
```

```javascript
// $lookup с pipeline (расширенный синтаксис)
db.orders.aggregate([
  { $lookup: {
    from: "products",
    let: { prodId: "$productId" },   // Переменные из текущего документа
    pipeline: [
      { $match: {
        $expr: { $eq: ["$_id", "$$prodId"] }  // $$ — доступ к переменной let
      }},
      { $project: { name: 1, price: 1 } }     // Только нужные поля
    ],
    as: "product"
  }}
])

// Множественный lookup
db.orders.aggregate([
  { $lookup: {
    from: "products",
    localField: "productId",
    foreignField: "_id",
    as: "product"
  }},
  { $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }},
  { $unwind: "$product" },
  { $unwind: "$customer" }
])
```

---

### $limit и $skip — Пагинация

```javascript
// Получить первые 10 документов
db.products.aggregate([
  { $sort: { createdAt: -1 } },
  { $limit: 10 }
])

// Пропустить первые 20, взять следующие 10 (страница 3)
db.products.aggregate([
  { $sort: { createdAt: -1 } },
  { $skip: 20 },
  { $limit: 10 }
])

// Пагинация с фильтрацией
const page = 2;
const pageSize = 10;

db.products.aggregate([
  { $match: { category: "electronics" } },
  { $sort: { price: -1 } },
  { $skip: (page - 1) * pageSize },
  { $limit: pageSize }
])
```

---

## Операторы аккумуляции

Операторы аккумуляции используются внутри `$group` для вычисления агрегированных значений.

### $sum — Сумма

```javascript
// Сумма значений поля
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    totalSpent: { $sum: "$amount" }
  }}
])

// Подсчёт количества документов
db.orders.aggregate([
  { $group: {
    _id: "$status",
    count: { $sum: 1 }
  }}
])

// Сумма с условием
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    completedTotal: {
      $sum: {
        $cond: [{ $eq: ["$status", "completed"] }, "$amount", 0]
      }
    }
  }}
])
```

### $avg — Среднее значение

```javascript
db.products.aggregate([
  { $group: {
    _id: "$category",
    avgPrice: { $avg: "$price" },
    avgRating: { $avg: "$rating" }
  }}
])
```

### $min и $max

```javascript
db.products.aggregate([
  { $group: {
    _id: "$category",
    cheapest: { $min: "$price" },
    mostExpensive: { $max: "$price" },
    firstAdded: { $min: "$createdAt" },
    lastAdded: { $max: "$createdAt" }
  }}
])
```

### $push — Сбор в массив

```javascript
// Собрать все значения в массив
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    orderIds: { $push: "$_id" },
    amounts: { $push: "$amount" }
  }}
])

// Собрать объекты
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    orders: { $push: { id: "$_id", amount: "$amount", date: "$date" } }
  }}
])

// $$ROOT — весь документ
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    allOrders: { $push: "$$ROOT" }
  }}
])
```

### $addToSet — Уникальные значения

```javascript
// Только уникальные значения (как Set)
db.orders.aggregate([
  { $group: {
    _id: "$customerId",
    uniqueProducts: { $addToSet: "$productId" },
    uniqueStatuses: { $addToSet: "$status" }
  }}
])
```

### $first и $last

```javascript
// Первый и последний документ в группе (после сортировки!)
db.orders.aggregate([
  { $sort: { date: 1 } },
  { $group: {
    _id: "$customerId",
    firstOrder: { $first: "$$ROOT" },
    lastOrder: { $last: "$$ROOT" },
    firstOrderDate: { $first: "$date" },
    lastOrderDate: { $last: "$date" }
  }}
])
```

### $count — Подсчёт

```javascript
// Общее количество документов
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $count: "completedOrders" }
])
// Результат: { completedOrders: 42 }
```

---

## $facet — Параллельные Pipeline

`$facet` позволяет выполнить несколько независимых pipeline параллельно и получить результаты в одном документе.

```javascript
// Многогранный анализ продуктов
db.products.aggregate([
  { $facet: {
    // Pipeline 1: Статистика по категориям
    "byCategory": [
      { $group: { _id: "$category", count: { $sum: 1 } } },
      { $sort: { count: -1 } }
    ],

    // Pipeline 2: Ценовая статистика
    "priceStats": [
      { $group: {
        _id: null,
        avgPrice: { $avg: "$price" },
        minPrice: { $min: "$price" },
        maxPrice: { $max: "$price" }
      }}
    ],

    // Pipeline 3: Топ-5 дорогих товаров
    "topExpensive": [
      { $sort: { price: -1 } },
      { $limit: 5 },
      { $project: { name: 1, price: 1 } }
    ],

    // Pipeline 4: Товары на распродаже
    "onSale": [
      { $match: { onSale: true } },
      { $count: "count" }
    ]
  }}
])

// Результат:
{
  byCategory: [
    { _id: "electronics", count: 150 },
    { _id: "clothing", count: 89 }
  ],
  priceStats: [
    { _id: null, avgPrice: 245.50, minPrice: 5, maxPrice: 2500 }
  ],
  topExpensive: [
    { name: "MacBook Pro", price: 2500 },
    { name: "iPhone", price: 1200 }
  ],
  onSale: [{ count: 34 }]
}
```

```javascript
// Пагинация с метаданными
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $facet: {
    "metadata": [
      { $count: "total" }
    ],
    "data": [
      { $sort: { price: -1 } },
      { $skip: 20 },
      { $limit: 10 }
    ]
  }}
])

// Результат:
{
  metadata: [{ total: 150 }],
  data: [ /* 10 продуктов */ ]
}
```

---

## Дополнительные стадии

### $addFields — Добавление полей

Похож на `$project`, но сохраняет все существующие поля.

```javascript
db.products.aggregate([
  { $addFields: {
    discountPrice: { $multiply: ["$price", 0.9] },
    isExpensive: { $gte: ["$price", 1000] }
  }}
])
```

### $set — Алиас для $addFields

```javascript
db.products.aggregate([
  { $set: {
    priceWithTax: { $multiply: ["$price", 1.2] }
  }}
])
```

### $unset — Удаление полей

```javascript
db.users.aggregate([
  { $unset: ["password", "internalNotes", "tempField"] }
])
```

### $replaceRoot — Замена корневого документа

```javascript
// Продвинуть вложенный объект на верхний уровень
db.users.aggregate([
  { $replaceRoot: { newRoot: "$profile" } }
])

// С объединением
db.users.aggregate([
  { $replaceRoot: {
    newRoot: { $mergeObjects: [{ _id: "$_id" }, "$profile"] }
  }}
])
```

### $bucket и $bucketAuto — Группировка по диапазонам

```javascript
// Ценовые диапазоны (ручные границы)
db.products.aggregate([
  { $bucket: {
    groupBy: "$price",
    boundaries: [0, 50, 100, 200, 500, 1000],
    default: "expensive",
    output: {
      count: { $sum: 1 },
      products: { $push: "$name" }
    }
  }}
])

// Автоматические диапазоны
db.products.aggregate([
  { $bucketAuto: {
    groupBy: "$price",
    buckets: 5,                    // Разбить на 5 групп
    output: {
      count: { $sum: 1 },
      avgPrice: { $avg: "$price" }
    }
  }}
])
```

### $sortByCount — Группировка и сортировка

Сокращение для `$group` + `$sort`.

```javascript
// Эквивалентно: $group + $sort по count
db.products.aggregate([
  { $sortByCount: "$category" }
])

// То же самое, но развёрнуто:
db.products.aggregate([
  { $group: { _id: "$category", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### $out и $merge — Сохранение результатов

```javascript
// Сохранить в новую коллекцию (перезаписывает!)
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $out: "customer_totals" }
])

// $merge — более гибкое сохранение (upsert)
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $merge: {
    into: "customer_stats",
    on: "_id",
    whenMatched: "merge",
    whenNotMatched: "insert"
  }}
])
```

---

## Оптимизация Aggregation запросов

### 1. Используйте индексы

```javascript
// $match и $sort в начале pipeline могут использовать индексы
db.orders.createIndex({ status: 1, date: -1 })

db.orders.aggregate([
  { $match: { status: "completed" } },  // Использует индекс
  { $sort: { date: -1 } },              // Использует индекс
  { $limit: 100 },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

### 2. Фильтруйте рано

```javascript
// ХОРОШО: $match в начале
db.orders.aggregate([
  { $match: { status: "completed", amount: { $gt: 100 } } },
  { $lookup: { from: "customers", ... } },
  { $project: { ... } }
])

// ПЛОХО: $match после $lookup
db.orders.aggregate([
  { $lookup: { from: "customers", ... } },
  { $match: { status: "completed" } },  // Обработает все документы перед фильтрацией
  { $project: { ... } }
])
```

### 3. Используйте $project для уменьшения размера документов

```javascript
// Убрать ненужные поля перед тяжёлыми операциями
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $project: { customerId: 1, amount: 1 } },  // Только нужные поля
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

### 4. Explain для анализа

```javascript
// Анализ плана выполнения
db.orders.explain("executionStats").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

### 5. Лимитируйте раньше

```javascript
// Если нужен только топ-10, лимитируйте до группировки где возможно
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $sort: { rating: -1 } },
  { $limit: 10 },                        // Лимит до тяжёлых операций
  { $lookup: { from: "reviews", ... } }
])
```

### 6. allowDiskUse для больших данных

```javascript
// Разрешить использование диска для операций > 100MB
db.orders.aggregate([
  { $group: { _id: "$product", total: { $sum: "$amount" } } }
], { allowDiskUse: true })
```

---

## Практические примеры

### Пример 1: Аналитика продаж

```javascript
// Коллекция sales
{
  _id: ObjectId(),
  date: ISODate("2024-01-15"),
  product: "Laptop",
  category: "electronics",
  quantity: 2,
  unitPrice: 1000,
  customerId: "c123",
  region: "Europe"
}

// Месячный отчёт по продажам
db.sales.aggregate([
  // Фильтр по году
  { $match: {
    date: {
      $gte: ISODate("2024-01-01"),
      $lt: ISODate("2025-01-01")
    }
  }},

  // Вычислить сумму продажи
  { $addFields: {
    saleAmount: { $multiply: ["$quantity", "$unitPrice"] }
  }},

  // Группировка по месяцу и категории
  { $group: {
    _id: {
      year: { $year: "$date" },
      month: { $month: "$date" },
      category: "$category"
    },
    totalSales: { $sum: "$saleAmount" },
    totalQuantity: { $sum: "$quantity" },
    ordersCount: { $sum: 1 },
    avgOrderValue: { $avg: "$saleAmount" }
  }},

  // Сортировка
  { $sort: { "_id.year": 1, "_id.month": 1, totalSales: -1 } },

  // Красивый вывод
  { $project: {
    _id: 0,
    period: {
      $concat: [
        { $toString: "$_id.year" },
        "-",
        { $cond: [{ $lt: ["$_id.month", 10] }, "0", ""] },
        { $toString: "$_id.month" }
      ]
    },
    category: "$_id.category",
    totalSales: { $round: ["$totalSales", 2] },
    totalQuantity: 1,
    ordersCount: 1,
    avgOrderValue: { $round: ["$avgOrderValue", 2] }
  }}
])
```

### Пример 2: Воронка конверсии

```javascript
// Коллекция user_events
{
  userId: "u123",
  event: "page_view",     // page_view → add_to_cart → checkout → purchase
  timestamp: ISODate(),
  page: "/products/laptop"
}

// Анализ воронки
db.user_events.aggregate([
  { $match: {
    timestamp: { $gte: ISODate("2024-01-01") }
  }},

  { $group: {
    _id: "$userId",
    events: { $addToSet: "$event" }
  }},

  { $project: {
    viewedProducts: { $in: ["page_view", "$events"] },
    addedToCart: { $in: ["add_to_cart", "$events"] },
    startedCheckout: { $in: ["checkout", "$events"] },
    completed: { $in: ["purchase", "$events"] }
  }},

  { $group: {
    _id: null,
    totalUsers: { $sum: 1 },
    viewed: { $sum: { $cond: ["$viewedProducts", 1, 0] } },
    addedToCart: { $sum: { $cond: ["$addedToCart", 1, 0] } },
    checkout: { $sum: { $cond: ["$startedCheckout", 1, 0] } },
    purchased: { $sum: { $cond: ["$completed", 1, 0] } }
  }},

  { $project: {
    _id: 0,
    funnel: {
      step1_view: "$viewed",
      step2_cart: "$addedToCart",
      step3_checkout: "$checkout",
      step4_purchase: "$purchased",
      conversionRate: {
        $multiply: [
          { $divide: ["$purchased", "$viewed"] },
          100
        ]
      }
    }
  }}
])
```

### Пример 3: Топ клиентов с деталями

```javascript
// Коллекции: customers, orders, products

db.orders.aggregate([
  // Фильтр по периоду
  { $match: {
    status: "completed",
    date: { $gte: ISODate("2024-01-01") }
  }},

  // Группировка по клиенту
  { $group: {
    _id: "$customerId",
    totalSpent: { $sum: "$amount" },
    ordersCount: { $sum: 1 },
    avgOrderValue: { $avg: "$amount" },
    lastOrderDate: { $max: "$date" },
    productIds: { $addToSet: "$productId" }
  }},

  // Топ-10
  { $sort: { totalSpent: -1 } },
  { $limit: 10 },

  // Подтягиваем данные клиента
  { $lookup: {
    from: "customers",
    localField: "_id",
    foreignField: "_id",
    as: "customerData"
  }},
  { $unwind: "$customerData" },

  // Финальная проекция
  { $project: {
    _id: 0,
    customerId: "$_id",
    name: "$customerData.name",
    email: "$customerData.email",
    totalSpent: { $round: ["$totalSpent", 2] },
    ordersCount: 1,
    avgOrderValue: { $round: ["$avgOrderValue", 2] },
    lastOrderDate: 1,
    uniqueProductsCount: { $size: "$productIds" }
  }}
])
```

### Пример 4: Анализ временных рядов

```javascript
// Статистика по дням недели и часам
db.orders.aggregate([
  { $match: {
    date: { $gte: ISODate("2024-01-01") }
  }},

  { $group: {
    _id: {
      dayOfWeek: { $dayOfWeek: "$date" },
      hour: { $hour: "$date" }
    },
    ordersCount: { $sum: 1 },
    totalRevenue: { $sum: "$amount" }
  }},

  { $addFields: {
    dayName: {
      $switch: {
        branches: [
          { case: { $eq: ["$_id.dayOfWeek", 1] }, then: "Воскресенье" },
          { case: { $eq: ["$_id.dayOfWeek", 2] }, then: "Понедельник" },
          { case: { $eq: ["$_id.dayOfWeek", 3] }, then: "Вторник" },
          { case: { $eq: ["$_id.dayOfWeek", 4] }, then: "Среда" },
          { case: { $eq: ["$_id.dayOfWeek", 5] }, then: "Четверг" },
          { case: { $eq: ["$_id.dayOfWeek", 6] }, then: "Пятница" },
          { case: { $eq: ["$_id.dayOfWeek", 7] }, then: "Суббота" }
        ],
        default: "Unknown"
      }
    }
  }},

  { $sort: { "_id.dayOfWeek": 1, "_id.hour": 1 } },

  { $project: {
    _id: 0,
    day: "$dayName",
    hour: "$_id.hour",
    orders: "$ordersCount",
    revenue: "$totalRevenue"
  }}
])
```

### Пример 5: Dashboard с $facet

```javascript
// Комплексный dashboard для e-commerce
db.orders.aggregate([
  { $match: {
    date: {
      $gte: ISODate("2024-01-01"),
      $lt: ISODate("2024-02-01")
    }
  }},

  { $facet: {
    // Общая статистика
    "summary": [
      { $group: {
        _id: null,
        totalRevenue: { $sum: "$amount" },
        totalOrders: { $sum: 1 },
        avgOrderValue: { $avg: "$amount" },
        uniqueCustomers: { $addToSet: "$customerId" }
      }},
      { $project: {
        totalRevenue: 1,
        totalOrders: 1,
        avgOrderValue: { $round: ["$avgOrderValue", 2] },
        uniqueCustomers: { $size: "$uniqueCustomers" }
      }}
    ],

    // Топ-5 товаров
    "topProducts": [
      { $group: {
        _id: "$productId",
        quantity: { $sum: "$quantity" },
        revenue: { $sum: "$amount" }
      }},
      { $sort: { revenue: -1 } },
      { $limit: 5 },
      { $lookup: {
        from: "products",
        localField: "_id",
        foreignField: "_id",
        as: "product"
      }},
      { $unwind: "$product" },
      { $project: {
        name: "$product.name",
        quantity: 1,
        revenue: 1
      }}
    ],

    // Распределение по статусам
    "byStatus": [
      { $group: {
        _id: "$status",
        count: { $sum: 1 },
        total: { $sum: "$amount" }
      }}
    ],

    // Тренд по дням
    "dailyTrend": [
      { $group: {
        _id: { $dateToString: { format: "%Y-%m-%d", date: "$date" } },
        orders: { $sum: 1 },
        revenue: { $sum: "$amount" }
      }},
      { $sort: { _id: 1 } }
    ],

    // Топ регионов
    "byRegion": [
      { $group: {
        _id: "$region",
        orders: { $sum: 1 },
        revenue: { $sum: "$amount" }
      }},
      { $sort: { revenue: -1 } },
      { $limit: 5 }
    ]
  }}
])
```

---

## Best Practices

### 1. Оптимизация порядка стадий

```javascript
// ПРАВИЛЬНО: $match → $sort → $limit → тяжёлые операции
db.collection.aggregate([
  { $match: { ... } },     // 1. Фильтруем (использует индекс)
  { $sort: { ... } },      // 2. Сортируем (использует индекс)
  { $limit: 100 },         // 3. Ограничиваем
  { $lookup: { ... } },    // 4. Тяжёлые операции
  { $project: { ... } }    // 5. Финальная проекция
])
```

### 2. Используйте индексы эффективно

```javascript
// Создайте составной индекс для частых aggregation запросов
db.orders.createIndex({ status: 1, date: -1, customerId: 1 })

// Проверяйте использование индексов
db.orders.explain().aggregate([...])
```

### 3. Избегайте $unwind где возможно

```javascript
// ИЗБЕГАЙТЕ: $unwind создаёт много документов
db.orders.aggregate([
  { $unwind: "$items" },
  { $group: { _id: "$items.productId", count: { $sum: 1 } } }
])

// ЛУЧШЕ: Используйте операторы массивов
db.orders.aggregate([
  { $project: {
    productCounts: {
      $map: {
        input: "$items",
        as: "item",
        in: { productId: "$$item.productId", qty: "$$item.quantity" }
      }
    }
  }}
])
```

### 4. Ограничивайте размер результата

```javascript
// Всегда используйте $limit для больших коллекций
db.logs.aggregate([
  { $match: { level: "error" } },
  { $sort: { timestamp: -1 } },
  { $limit: 1000 }           // Защита от слишком большого результата
])
```

### 5. Модульность через переменные

```javascript
// Выносите стадии в переменные для читаемости
const matchStage = { $match: { status: "active" } };
const groupStage = { $group: { _id: "$category", count: { $sum: 1 } } };
const sortStage = { $sort: { count: -1 } };

db.products.aggregate([matchStage, groupStage, sortStage]);
```

### 6. Обработка null и отсутствующих полей

```javascript
// Используйте $ifNull для значений по умолчанию
db.products.aggregate([
  { $project: {
    name: 1,
    price: { $ifNull: ["$price", 0] },
    category: { $ifNull: ["$category", "uncategorized"] }
  }}
])
```

### 7. Мониторинг производительности

```javascript
// Используйте explain для анализа
const result = db.orders.explain("executionStats").aggregate([...]);

// Ключевые метрики:
// - executionTimeMillis
// - totalDocsExamined
// - indexesUsed
```

---

## Полезные ссылки

- [MongoDB Aggregation Pipeline Operators](https://www.mongodb.com/docs/manual/reference/operator/aggregation/)
- [Aggregation Pipeline Optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)
- [Aggregation Pipeline Quick Reference](https://www.mongodb.com/docs/manual/meta/aggregation-quick-reference/)
