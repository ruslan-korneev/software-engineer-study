# Модель данных и типы в MongoDB

## BSON — Binary JSON

**BSON** (Binary JSON) — это бинарный формат сериализации, используемый MongoDB для хранения документов и передачи данных.

### Почему BSON, а не JSON?

| Характеристика | JSON | BSON |
|----------------|------|------|
| Формат | Текстовый | Бинарный |
| Размер | Компактнее для простых данных | Больше метаданных |
| Парсинг | Медленнее | Быстрее |
| Типы данных | Ограничены | Расширенные |
| Обход | Требует полного парсинга | Можно пропускать поля |

### Преимущества BSON

1. **Скорость**: бинарный формат быстрее парсится
2. **Типизация**: поддержка дополнительных типов (Date, Binary, ObjectId)
3. **Обход**: можно быстро перемещаться по документу без полного парсинга
4. **Размер**: хранит длину строк и документов для быстрого доступа

### Пример: JSON vs BSON

```json
// JSON (текст, 44 байта)
{"name": "Иван", "age": 30, "active": true}

// BSON (бинарный, ~50 байт)
\x32\x00\x00\x00                         // размер документа
\x02 name\x00 \x09\x00\x00\x00 Иван\x00  // string
\x10 age\x00 \x1e\x00\x00\x00            // int32
\x08 active\x00 \x01                     // boolean
\x00                                     // конец документа
```

---

## Типы данных MongoDB

### Основные типы

| Тип | BSON Type | Описание |
|-----|-----------|----------|
| Double | 1 | 64-bit IEEE 754 floating point |
| String | 2 | UTF-8 строка |
| Object | 3 | Вложенный документ |
| Array | 4 | Массив значений |
| Binary | 5 | Бинарные данные |
| ObjectId | 7 | 12-байтный идентификатор |
| Boolean | 8 | true/false |
| Date | 9 | UTC datetime (миллисекунды с Unix epoch) |
| Null | 10 | Null значение |
| RegExp | 11 | Регулярное выражение |
| Int32 | 16 | 32-bit integer |
| Timestamp | 17 | Специальный тип для репликации |
| Int64 | 18 | 64-bit integer |
| Decimal128 | 19 | 128-bit decimal (высокая точность) |

---

## ObjectId

**ObjectId** — это 12-байтный уникальный идентификатор, используемый по умолчанию для поля `_id`.

### Структура ObjectId

```
|-------- 12 байт --------|
| 4 байта | 5 байт | 3 байта |
| timestamp| random | counter |

Пример: 507f1f77bcf86cd799439011

507f1f77  - timestamp (секунды с Unix epoch)
bcf86cd799 - случайное значение (machine + process)
439011    - инкрементный счётчик
```

### Работа с ObjectId

```javascript
// Создание нового ObjectId
const id = new ObjectId();

// Создание из строки
const id = new ObjectId("507f1f77bcf86cd799439011");

// Получение timestamp
id.getTimestamp();  // ISODate("2012-10-17T20:46:22Z")

// Сравнение
id1.equals(id2);

// Преобразование в строку
id.toString();  // "507f1f77bcf86cd799439011"
id.toHexString();
```

### Преимущества ObjectId

- **Уникальность**: гарантированно уникален в распределённой системе
- **Сортируемость**: грубо упорядочен по времени создания
- **Встроенный timestamp**: можно извлечь время создания
- **Компактность**: 12 байт vs 36 байт для UUID

---

## String (Строки)

Строки в MongoDB хранятся в формате UTF-8.

```javascript
// Обычная строка
{ name: "Привет, мир!" }

// Многострочная строка
{
  description: `Первая строка
  Вторая строка
  Третья строка`
}

// Операции со строками в запросах
db.users.find({ name: /^Иван/i })  // регулярное выражение
db.users.find({ name: { $regex: "иван", $options: "i" } })
```

---

## Numbers (Числа)

### Типы чисел

```javascript
// По умолчанию в shell — Double
{ price: 99.99 }

// Явное указание Int32
{ count: NumberInt(100) }

// Явное указание Int64
{ views: NumberLong("9007199254740993") }

// Decimal128 для финансовых расчётов
{ amount: NumberDecimal("19.99") }
```

### Проблема точности Double

```javascript
// Проблема с Double
0.1 + 0.2  // 0.30000000000000004

// Решение: Decimal128
NumberDecimal("0.1") + NumberDecimal("0.2")  // "0.3"

// Или хранить в минимальных единицах (копейки)
{ price_cents: 9999 }  // 99.99 рублей
```

---

## Date (Даты)

MongoDB хранит даты как 64-bit integer (миллисекунды с 1 января 1970 UTC).

```javascript
// Текущая дата
{ created_at: new Date() }

// Конкретная дата
{ birthday: new Date("1990-05-15") }
{ birthday: ISODate("1990-05-15T10:30:00Z") }

// Создание из компонентов
{ date: new Date(1990, 4, 15) }  // месяцы 0-11!

// Timestamp (миллисекунды)
{ timestamp: new Date().getTime() }  // 1703673600000
```

### Операции с датами

```javascript
// Поиск по диапазону дат
db.events.find({
  date: {
    $gte: ISODate("2024-01-01"),
    $lt: ISODate("2024-02-01")
  }
})

// Агрегация по дате
db.orders.aggregate([
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$created_at" } },
      count: { $sum: 1 }
    }
  }
])
```

---

## Boolean

```javascript
// Булевые значения
{ active: true, verified: false }

// В запросах
db.users.find({ active: true })
db.users.find({ verified: { $ne: true } })  // false или отсутствует
```

---

## Null

```javascript
// Null значение
{ middleName: null }

// Поиск null
db.users.find({ middleName: null })  // null или отсутствует

// Поиск только null (не отсутствующие)
db.users.find({ middleName: { $type: "null" } })

// Поиск отсутствующих полей
db.users.find({ middleName: { $exists: false } })
```

---

## Array (Массивы)

Массивы могут содержать значения любых типов, включая другие массивы и документы.

```javascript
// Массив примитивов
{ tags: ["mongodb", "database", "nosql"] }

// Массив чисел
{ scores: [85, 90, 78, 92] }

// Массив документов
{
  orders: [
    { product: "Книга", price: 500 },
    { product: "Ручка", price: 50 }
  ]
}

// Смешанный массив (не рекомендуется)
{ mixed: ["text", 123, true, { key: "value" }] }
```

### Операции с массивами

```javascript
// Поиск по элементу массива
db.posts.find({ tags: "mongodb" })

// Поиск по всем элементам
db.posts.find({ tags: { $all: ["mongodb", "nosql"] } })

// Поиск по размеру
db.posts.find({ tags: { $size: 3 } })

// Поиск в массиве документов
db.orders.find({ "items.product": "Книга" })

// $elemMatch — все условия для одного элемента
db.orders.find({
  items: { $elemMatch: { product: "Книга", quantity: { $gte: 2 } } }
})
```

---

## Embedded Documents (Вложенные документы)

```javascript
// Вложенный документ
{
  name: "Иван Петров",
  address: {
    city: "Москва",
    street: "Тверская",
    building: 15,
    apartment: 42
  },
  contacts: {
    email: "ivan@example.com",
    phone: "+7 999 123-45-67"
  }
}
```

### Доступ к вложенным полям

```javascript
// Dot notation
db.users.find({ "address.city": "Москва" })

// Обновление вложенного поля
db.users.updateOne(
  { name: "Иван Петров" },
  { $set: { "address.apartment": 43 } }
)

// Обновление всего вложенного документа
db.users.updateOne(
  { name: "Иван Петров" },
  { $set: { address: { city: "Санкт-Петербург", street: "Невский" } } }
)
```

---

## Binary Data

Бинарные данные для хранения файлов, изображений и других бинарных объектов.

```javascript
// Создание бинарных данных
{
  avatar: new BinData(0, "base64encodedstring..."),
  avatarType: "image/png"
}

// Subtypes
// 0 - Generic binary
// 1 - Function
// 2 - Binary (old)
// 3 - UUID (old)
// 4 - UUID
// 5 - MD5
// 128-255 - User defined
```

> **Рекомендация**: для больших файлов (>16MB) используйте GridFS.

---

## Embedded Documents vs References

### Подход 1: Embedded (Встраивание)

```javascript
// Всё в одном документе
{
  _id: ObjectId("..."),
  title: "Как изучать MongoDB",
  author: {
    name: "Анна Иванова",
    email: "anna@example.com"
  },
  comments: [
    { user: "Петр", text: "Отличная статья!", date: ISODate("...") },
    { user: "Мария", text: "Спасибо!", date: ISODate("...") }
  ]
}
```

**Плюсы:**
- Один запрос для получения всех данных
- Атомарные операции
- Высокая производительность чтения

**Минусы:**
- Дублирование данных
- Ограничение 16MB на документ
- Сложно обновлять общие данные

### Подход 2: References (Ссылки)

```javascript
// Коллекция posts
{
  _id: ObjectId("post1"),
  title: "Как изучать MongoDB",
  author_id: ObjectId("user1"),
  comment_ids: [ObjectId("comment1"), ObjectId("comment2")]
}

// Коллекция users
{
  _id: ObjectId("user1"),
  name: "Анна Иванова",
  email: "anna@example.com"
}

// Коллекция comments
{
  _id: ObjectId("comment1"),
  post_id: ObjectId("post1"),
  user: "Петр",
  text: "Отличная статья!"
}
```

**Плюсы:**
- Нет дублирования
- Нет ограничения на количество связанных документов
- Легко обновлять общие данные

**Минусы:**
- Требуется $lookup или несколько запросов
- Нет атомарности между коллекциями
- Сложнее поддерживать консистентность

### Когда что использовать?

| Критерий | Embedding | References |
|----------|-----------|------------|
| Отношение | One-to-Few | One-to-Many, Many-to-Many |
| Частота чтения вместе | Всегда | Редко |
| Размер вложенных данных | Маленький | Большой/неограниченный |
| Обновления | Независимые | Частые обновления связанных данных |

---

## Schema Design Patterns

### 1. Attribute Pattern

Для документов с разнообразными характеристиками.

```javascript
// Плохо: разные поля для разных товаров
{
  product: "Ноутбук",
  screen_size: "15.6",
  ram: "16GB",
  storage: "512GB SSD"
}
{
  product: "Футболка",
  size: "M",
  color: "синий",
  material: "хлопок"
}

// Хорошо: универсальный массив атрибутов
{
  product: "Ноутбук",
  attributes: [
    { k: "screen_size", v: "15.6", u: "дюймов" },
    { k: "ram", v: "16", u: "GB" },
    { k: "storage", v: "512", u: "GB SSD" }
  ]
}
```

### 2. Bucket Pattern

Группировка time-series данных.

```javascript
// Плохо: документ на каждое измерение
{ sensor_id: 1, temp: 22.5, time: ISODate("2024-01-01T10:00:00") }
{ sensor_id: 1, temp: 22.7, time: ISODate("2024-01-01T10:01:00") }
// ... миллионы документов

// Хорошо: группировка по часам
{
  sensor_id: 1,
  date: ISODate("2024-01-01T10:00:00"),
  measurements: [
    { minute: 0, temp: 22.5 },
    { minute: 1, temp: 22.7 },
    // ... 60 измерений
  ],
  count: 60,
  avg_temp: 22.6
}
```

### 3. Extended Reference Pattern

Встраивание часто используемых полей из связанных документов.

```javascript
// orders collection
{
  _id: ObjectId("..."),
  date: ISODate("..."),
  customer_id: ObjectId("customer1"),
  // Копия основных полей клиента
  customer_name: "Иван Петров",
  customer_email: "ivan@example.com",
  items: [...]
}
```

### 4. Subset Pattern

Встраивание только последних/популярных элементов.

```javascript
// Коллекция movies
{
  _id: ObjectId("..."),
  title: "Начало",
  // Только последние 10 отзывов
  recent_reviews: [
    { user: "Анна", rating: 5, text: "Шедевр!", date: ISODate("...") },
    // ...
  ],
  total_reviews: 15847,
  avg_rating: 4.8
}

// Коллекция reviews — все отзывы
{
  movie_id: ObjectId("..."),
  user: "Анна",
  rating: 5,
  text: "Шедевр!",
  date: ISODate("...")
}
```

---

## Ограничение размера документа

- **Максимальный размер документа**: 16 MB
- **Максимальная вложенность**: 100 уровней

```javascript
// Проверка размера документа
Object.bsonsize(db.users.findOne({ name: "Иван" }))

// Или в shell
bsonsize(db.users.findOne())
```

### Что делать при достижении лимита?

1. **GridFS**: для файлов > 16MB
2. **Bucket Pattern**: группировка time-series данных
3. **References**: разделение на связанные коллекции
4. **Subset Pattern**: хранение только части данных

---

## Best Practices для модели данных

1. **Проектируйте схему под запросы**: оптимизируйте структуру для частых операций чтения
2. **Встраивайте данные, которые читаются вместе**: один запрос вместо нескольких
3. **Используйте references для часто обновляемых данных**: избегайте дублирования
4. **Денормализуйте для производительности**: копируйте часто читаемые поля
5. **Следите за размером документа**: не приближайтесь к лимиту 16MB
6. **Используйте осмысленные имена полей**: но помните, что они хранятся в каждом документе
7. **Индексируйте поля для запросов**: особенно вложенные поля и массивы
