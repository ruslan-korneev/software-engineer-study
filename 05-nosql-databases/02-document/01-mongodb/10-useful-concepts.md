# Полезные концепции MongoDB

MongoDB предоставляет множество продвинутых возможностей, которые расширяют базовый функционал документоориентированной базы данных. В этом разделе рассмотрим ключевые концепции, которые помогают решать специализированные задачи.

---

## 1. Change Streams (Потоки изменений)

Change Streams позволяют приложениям подписываться на изменения данных в реальном времени. Это реактивный подход к обработке изменений в базе данных.

### Основные возможности

- Получение уведомлений о вставках, обновлениях, удалениях и замене документов
- Фильтрация изменений на уровне базы данных
- Возобновление потока с определённой точки (resume token)
- Работает только с replica set или sharded cluster

### Примеры использования

```javascript
// Базовая подписка на изменения коллекции
const changeStream = db.collection('orders').watch();

changeStream.on('change', (change) => {
    console.log('Тип операции:', change.operationType);
    console.log('Документ:', change.fullDocument);
});

// Подписка с фильтрацией
const pipeline = [
    { $match: { 'fullDocument.status': 'pending' } },
    { $project: { documentKey: 1, fullDocument: 1 } }
];

const filteredStream = db.collection('orders').watch(pipeline);

// Подписка на всю базу данных
const dbChangeStream = db.watch();

// Возобновление потока с определённой точки
const resumeToken = { _data: 'saved_token...' };
const resumedStream = db.collection('orders').watch([], {
    resumeAfter: resumeToken
});
```

### Python пример (pymongo)

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')
db = client.myapp

# Подписка на изменения
with db.orders.watch() as stream:
    for change in stream:
        print(f"Операция: {change['operationType']}")
        if 'fullDocument' in change:
            print(f"Документ: {change['fullDocument']}")
```

### Практическое применение

- Синхронизация данных между системами
- Real-time уведомления пользователям
- Инвалидация кэша
- Event-driven архитектура
- Аудит изменений

---

## 2. GridFS (Хранение больших файлов)

GridFS — это спецификация для хранения и извлечения файлов, превышающих лимит BSON-документа в 16 МБ. Файлы разбиваются на части (chunks) и хранятся в двух коллекциях.

### Структура GridFS

```
fs.files     — метаданные файлов
fs.chunks   — части файлов (по умолчанию 255 КБ каждая)
```

### Схема документа fs.files

```javascript
{
    _id: ObjectId("..."),
    length: 1048576,              // размер файла в байтах
    chunkSize: 261120,            // размер chunk в байтах
    uploadDate: ISODate("..."),   // дата загрузки
    filename: "video.mp4",        // имя файла
    metadata: {                    // пользовательские метаданные
        contentType: "video/mp4",
        author: "user123"
    }
}
```

### Примеры работы с GridFS

```javascript
// Node.js с mongodb driver
const { MongoClient, GridFSBucket } = require('mongodb');
const fs = require('fs');

async function uploadFile() {
    const client = await MongoClient.connect('mongodb://localhost:27017');
    const db = client.db('myapp');

    const bucket = new GridFSBucket(db, {
        bucketName: 'uploads',     // имя bucket (по умолчанию 'fs')
        chunkSizeBytes: 1048576    // 1 МБ на chunk
    });

    // Загрузка файла
    const uploadStream = bucket.openUploadStream('large-video.mp4', {
        metadata: {
            contentType: 'video/mp4',
            uploadedBy: 'user123'
        }
    });

    fs.createReadStream('./video.mp4')
        .pipe(uploadStream)
        .on('finish', () => console.log('Файл загружен'));

    // Скачивание файла
    const downloadStream = bucket.openDownloadStreamByName('large-video.mp4');
    downloadStream.pipe(fs.createWriteStream('./downloaded-video.mp4'));

    // Удаление файла
    await bucket.delete(fileId);

    // Поиск файлов
    const files = await bucket.find({
        'metadata.contentType': 'video/mp4'
    }).toArray();
}
```

### Python пример (gridfs)

```python
from pymongo import MongoClient
import gridfs

client = MongoClient('mongodb://localhost:27017/')
db = client.myapp
fs = gridfs.GridFS(db)

# Загрузка файла
with open('large_file.pdf', 'rb') as f:
    file_id = fs.put(f, filename='document.pdf',
                     content_type='application/pdf',
                     author='user123')

# Скачивание файла
grid_out = fs.get(file_id)
with open('downloaded.pdf', 'wb') as f:
    f.write(grid_out.read())

# Поиск файлов
for grid_file in fs.find({'filename': {'$regex': r'\.pdf$'}}):
    print(f"{grid_file.filename}: {grid_file.length} bytes")

# Удаление файла
fs.delete(file_id)
```

### Когда использовать GridFS

**Подходит для:**
- Файлы больше 16 МБ
- Доступ к частям файла без загрузки целиком
- Хранение метаданных вместе с файлами
- Репликация файлов вместе с данными

**Не подходит для:**
- Маленьких файлов (лучше хранить как Binary Data в документе)
- Частого изменения файлов (каждое изменение = перезапись)
- Случаев, когда нужна файловая система (используйте S3, MinIO)

---

## 3. Capped Collections (Коллекции фиксированного размера)

Capped Collections — это коллекции с фиксированным размером, которые работают по принципу кольцевого буфера. Когда коллекция заполняется, старые документы автоматически удаляются.

### Характеристики

- Фиксированный размер в байтах
- Опциональное ограничение количества документов
- Порядок вставки гарантирован (natural order)
- Высокая производительность записи
- Нельзя удалять отдельные документы
- Нельзя изменять размер документа (если он увеличится)

### Создание Capped Collection

```javascript
// Создание с ограничением по размеру
db.createCollection('logs', {
    capped: true,
    size: 104857600       // 100 МБ максимальный размер
});

// Создание с ограничением по размеру и количеству документов
db.createCollection('events', {
    capped: true,
    size: 52428800,       // 50 МБ
    max: 10000            // максимум 10000 документов
});

// Проверка, является ли коллекция capped
db.logs.isCapped();

// Конвертация обычной коллекции в capped
db.runCommand({
    convertToCapped: 'myCollection',
    size: 10485760        // 10 МБ
});
```

### Работа с Capped Collections

```javascript
// Вставка (как обычно)
db.logs.insertOne({
    timestamp: new Date(),
    level: 'INFO',
    message: 'Application started'
});

// Чтение в порядке вставки (natural order)
db.logs.find().sort({ $natural: 1 });

// Чтение в обратном порядке
db.logs.find().sort({ $natural: -1 });

// Tailable Cursor — получение новых документов в реальном времени
const cursor = db.logs.find({}, {
    tailable: true,
    awaitData: true
});

// В Node.js
const cursor = collection.find({}).tailable().awaitData();
for await (const doc of cursor) {
    console.log(doc);
}
```

### Практическое применение

```javascript
// Логирование приложения
db.createCollection('app_logs', {
    capped: true,
    size: 536870912    // 512 МБ для логов
});

// Очередь сообщений
db.createCollection('message_queue', {
    capped: true,
    size: 104857600,
    max: 100000        // максимум 100k сообщений
});

// Кэш последних действий пользователя
db.createCollection('user_activity', {
    capped: true,
    size: 10485760,
    max: 1000          // последние 1000 действий
});
```

---

## 4. TTL индексы (Time-To-Live)

TTL индексы автоматически удаляют документы по истечении определённого времени. Это полезно для данных с ограниченным сроком жизни.

### Создание TTL индекса

```javascript
// Удаление через 3600 секунд (1 час) после значения поля createdAt
db.sessions.createIndex(
    { createdAt: 1 },
    { expireAfterSeconds: 3600 }
);

// Документ будет удалён через 1 час после createdAt
db.sessions.insertOne({
    userId: 'user123',
    token: 'abc123',
    createdAt: new Date()
});

// TTL с конкретной датой истечения
db.tokens.createIndex(
    { expiresAt: 1 },
    { expireAfterSeconds: 0 }  // удаление в момент expiresAt
);

db.tokens.insertOne({
    token: 'xyz789',
    expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000)  // через 24 часа
});
```

### Особенности TTL индексов

```javascript
// TTL работает только с полями типа Date или массивом дат
// Для массива используется самая ранняя дата

// Фоновый процесс удаления запускается каждые 60 секунд
// Удаление не мгновенное!

// TTL индекс на составном поле НЕ работает
// Это НЕ будет работать как TTL:
db.data.createIndex({ createdAt: 1, status: 1 }, { expireAfterSeconds: 3600 });

// Изменение времени жизни существующего TTL индекса
db.runCommand({
    collMod: 'sessions',
    index: {
        keyPattern: { createdAt: 1 },
        expireAfterSeconds: 7200  // изменяем на 2 часа
    }
});
```

### Практические примеры

```javascript
// Сессии пользователей (истекают через 30 дней)
db.createCollection('user_sessions');
db.user_sessions.createIndex(
    { lastActivity: 1 },
    { expireAfterSeconds: 2592000 }  // 30 дней
);

// Временные данные верификации (истекают через 15 минут)
db.verification_codes.createIndex(
    { createdAt: 1 },
    { expireAfterSeconds: 900 }
);

// Кэш с конкретным временем истечения
db.cache.createIndex(
    { expiresAt: 1 },
    { expireAfterSeconds: 0 }
);

db.cache.insertOne({
    key: 'user:123:profile',
    data: { name: 'John', email: 'john@example.com' },
    expiresAt: new Date(Date.now() + 5 * 60 * 1000)  // 5 минут
});
```

---

## 5. Text Search (Полнотекстовый поиск)

MongoDB поддерживает полнотекстовый поиск с учётом языковых особенностей, стоп-слов и стемминга.

### Создание текстового индекса

```javascript
// Индекс на одно поле
db.articles.createIndex({ content: 'text' });

// Индекс на несколько полей с весами
db.articles.createIndex(
    {
        title: 'text',
        content: 'text',
        tags: 'text'
    },
    {
        weights: {
            title: 10,      // заголовок важнее
            content: 5,
            tags: 3
        },
        name: 'TextIndex',
        default_language: 'russian'
    }
);

// Wildcard text index (все строковые поля)
db.products.createIndex({ '$**': 'text' });
```

### Выполнение текстового поиска

```javascript
// Базовый поиск
db.articles.find({ $text: { $search: 'mongodb базы данных' } });

// Поиск фразы (в кавычках)
db.articles.find({ $text: { $search: '"NoSQL база данных"' } });

// Исключение слова
db.articles.find({ $text: { $search: 'mongodb -mysql' } });

// Поиск с указанием языка
db.articles.find({
    $text: {
        $search: 'программирование',
        $language: 'russian'
    }
});

// Поиск без учёта диакритических знаков
db.articles.find({
    $text: {
        $search: 'cafe',
        $diacriticSensitive: false
    }
});

// Сортировка по релевантности
db.articles.find(
    { $text: { $search: 'mongodb tutorial' } },
    { score: { $meta: 'textScore' } }
).sort({ score: { $meta: 'textScore' } });
```

### Агрегация с текстовым поиском

```javascript
db.articles.aggregate([
    { $match: { $text: { $search: 'mongodb' } } },
    { $addFields: { score: { $meta: 'textScore' } } },
    { $match: { score: { $gt: 1.0 } } },  // фильтр по релевантности
    { $sort: { score: -1 } },
    { $limit: 10 },
    { $project: { title: 1, score: 1 } }
]);
```

### Ограничения текстового поиска

- Только один текстовый индекс на коллекцию
- Не поддерживает частичное совпадение (wildcard) — ищет целые слова
- Для сложного поиска лучше использовать Atlas Search или Elasticsearch

---

## 6. Geospatial Queries (Геопространственные запросы)

MongoDB поддерживает хранение и запросы географических данных в форматах GeoJSON и legacy coordinate pairs.

### Типы геопространственных данных (GeoJSON)

```javascript
// Point (точка)
{
    type: 'Point',
    coordinates: [longitude, latitude]  // [долгота, широта]
}

// LineString (линия)
{
    type: 'LineString',
    coordinates: [[lng1, lat1], [lng2, lat2], [lng3, lat3]]
}

// Polygon (полигон)
{
    type: 'Polygon',
    coordinates: [[[lng1, lat1], [lng2, lat2], [lng3, lat3], [lng1, lat1]]]
}
```

### Создание геопространственных индексов

```javascript
// 2dsphere индекс для GeoJSON (сферическая геометрия)
db.places.createIndex({ location: '2dsphere' });

// 2d индекс для плоской геометрии (legacy)
db.oldPlaces.createIndex({ coordinates: '2d' });
```

### Вставка геоданных

```javascript
db.restaurants.insertMany([
    {
        name: 'Кафе Пушкинъ',
        location: {
            type: 'Point',
            coordinates: [37.6042, 55.7651]  // Москва
        },
        cuisine: 'Russian'
    },
    {
        name: 'Теремок',
        location: {
            type: 'Point',
            coordinates: [37.6173, 55.7558]
        },
        cuisine: 'Russian Fast Food'
    }
]);
```

### Геопространственные запросы

```javascript
// Поиск рядом с точкой
db.restaurants.find({
    location: {
        $near: {
            $geometry: {
                type: 'Point',
                coordinates: [37.6176, 55.7520]
            },
            $maxDistance: 1000,  // в метрах
            $minDistance: 100
        }
    }
});

// Поиск в радиусе (возвращает расстояние)
db.restaurants.aggregate([
    {
        $geoNear: {
            near: { type: 'Point', coordinates: [37.6176, 55.7520] },
            distanceField: 'distance',
            maxDistance: 2000,
            spherical: true
        }
    }
]);

// Поиск внутри полигона
db.restaurants.find({
    location: {
        $geoWithin: {
            $geometry: {
                type: 'Polygon',
                coordinates: [[
                    [37.60, 55.75],
                    [37.62, 55.75],
                    [37.62, 55.77],
                    [37.60, 55.77],
                    [37.60, 55.75]
                ]]
            }
        }
    }
});

// Поиск внутри круга
db.restaurants.find({
    location: {
        $geoWithin: {
            $centerSphere: [[37.6176, 55.7520], 1 / 6378.1]  // 1 км радиус
        }
    }
});

// Пересечение с геометрией
db.areas.find({
    geometry: {
        $geoIntersects: {
            $geometry: {
                type: 'LineString',
                coordinates: [[37.60, 55.75], [37.65, 55.78]]
            }
        }
    }
});
```

### Практический пример: сервис доставки

```javascript
// Создание коллекции ресторанов с зонами доставки
db.restaurants.insertOne({
    name: 'Пиццерия',
    location: { type: 'Point', coordinates: [37.6176, 55.7520] },
    deliveryZone: {
        type: 'Polygon',
        coordinates: [[
            [37.60, 55.74], [37.64, 55.74],
            [37.64, 55.76], [37.60, 55.76],
            [37.60, 55.74]
        ]]
    }
});

db.restaurants.createIndex({ deliveryZone: '2dsphere' });

// Проверка, доставляет ли ресторан по адресу
db.restaurants.find({
    deliveryZone: {
        $geoIntersects: {
            $geometry: {
                type: 'Point',
                coordinates: [37.62, 55.75]  // адрес клиента
            }
        }
    }
});
```

---

## 7. Schema Validation (Валидация схемы)

Schema Validation позволяет задать правила валидации документов при вставке и обновлении.

### Создание коллекции с валидацией

```javascript
db.createCollection('users', {
    validator: {
        $jsonSchema: {
            bsonType: 'object',
            required: ['username', 'email', 'password'],
            properties: {
                username: {
                    bsonType: 'string',
                    minLength: 3,
                    maxLength: 50,
                    description: 'Имя пользователя (3-50 символов)'
                },
                email: {
                    bsonType: 'string',
                    pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
                    description: 'Валидный email адрес'
                },
                password: {
                    bsonType: 'string',
                    minLength: 8,
                    description: 'Пароль минимум 8 символов'
                },
                age: {
                    bsonType: 'int',
                    minimum: 0,
                    maximum: 150,
                    description: 'Возраст от 0 до 150'
                },
                status: {
                    enum: ['active', 'inactive', 'banned'],
                    description: 'Статус пользователя'
                },
                roles: {
                    bsonType: 'array',
                    items: {
                        bsonType: 'string',
                        enum: ['user', 'admin', 'moderator']
                    }
                },
                profile: {
                    bsonType: 'object',
                    properties: {
                        firstName: { bsonType: 'string' },
                        lastName: { bsonType: 'string' },
                        avatar: { bsonType: 'string' }
                    }
                },
                createdAt: {
                    bsonType: 'date'
                }
            },
            additionalProperties: false  // запретить дополнительные поля
        }
    },
    validationLevel: 'strict',      // strict | moderate
    validationAction: 'error'       // error | warn
});
```

### Уровни и действия валидации

```javascript
// validationLevel:
// - 'strict' — валидация применяется ко всем вставкам и обновлениям
// - 'moderate' — валидация только для документов, которые уже соответствуют схеме

// validationAction:
// - 'error' — отклонить невалидный документ
// - 'warn' — разрешить, но записать предупреждение в лог
```

### Изменение валидации существующей коллекции

```javascript
db.runCommand({
    collMod: 'users',
    validator: {
        $jsonSchema: {
            bsonType: 'object',
            required: ['username', 'email'],
            properties: {
                username: { bsonType: 'string', minLength: 3 },
                email: { bsonType: 'string' }
            }
        }
    },
    validationLevel: 'moderate',
    validationAction: 'warn'
});
```

### Валидация с использованием query operators

```javascript
db.createCollection('orders', {
    validator: {
        $and: [
            { status: { $in: ['pending', 'processing', 'shipped', 'delivered'] } },
            { totalAmount: { $gt: 0 } },
            { items: { $type: 'array' } },
            { 'items.0': { $exists: true } }  // минимум 1 товар
        ]
    }
});
```

### Практический пример

```javascript
// Создание коллекции продуктов с полной валидацией
db.createCollection('products', {
    validator: {
        $jsonSchema: {
            bsonType: 'object',
            required: ['name', 'price', 'category'],
            properties: {
                name: {
                    bsonType: 'string',
                    minLength: 1,
                    maxLength: 200
                },
                price: {
                    bsonType: 'decimal',
                    minimum: 0
                },
                category: {
                    bsonType: 'string',
                    enum: ['electronics', 'clothing', 'food', 'books']
                },
                stock: {
                    bsonType: 'int',
                    minimum: 0
                },
                attributes: {
                    bsonType: 'object',
                    additionalProperties: true
                },
                tags: {
                    bsonType: 'array',
                    uniqueItems: true,
                    items: { bsonType: 'string' }
                }
            }
        }
    }
});
```

---

## 8. Views (Представления)

Views — это виртуальные коллекции, определяемые агрегационным пайплайном. Они не хранят данные, а вычисляют результат при каждом запросе.

### Создание View

```javascript
// Простой view — только активные пользователи
db.createView(
    'activeUsers',           // имя view
    'users',                 // исходная коллекция
    [{ $match: { status: 'active' } }]  // pipeline
);

// Сложный view с join и трансформацией
db.createView(
    'orderSummary',
    'orders',
    [
        { $match: { status: 'completed' } },
        { $lookup: {
            from: 'users',
            localField: 'userId',
            foreignField: '_id',
            as: 'user'
        }},
        { $unwind: '$user' },
        { $project: {
            orderId: '$_id',
            customerName: '$user.name',
            customerEmail: '$user.email',
            totalAmount: 1,
            itemCount: { $size: '$items' },
            orderDate: '$createdAt'
        }}
    ]
);
```

### Работа с View

```javascript
// Использование как обычной коллекции
db.activeUsers.find({ age: { $gte: 18 } });
db.activeUsers.countDocuments();

// Агрегация поверх view
db.orderSummary.aggregate([
    { $group: {
        _id: '$customerEmail',
        totalSpent: { $sum: '$totalAmount' },
        orderCount: { $sum: 1 }
    }}
]);
```

### View с параметрами (через системную переменную)

```javascript
// Использование $$NOW для текущего времени
db.createView(
    'recentOrders',
    'orders',
    [
        { $match: {
            $expr: {
                $gte: [
                    '$createdAt',
                    { $subtract: ['$$NOW', 7 * 24 * 60 * 60 * 1000] }  // последние 7 дней
                ]
            }
        }}
    ]
);
```

### Практические примеры

```javascript
// View для аналитики продаж
db.createView(
    'salesAnalytics',
    'orders',
    [
        { $match: { status: 'completed' } },
        { $unwind: '$items' },
        { $group: {
            _id: {
                productId: '$items.productId',
                month: { $month: '$createdAt' },
                year: { $year: '$createdAt' }
            },
            totalQuantity: { $sum: '$items.quantity' },
            totalRevenue: { $sum: { $multiply: ['$items.price', '$items.quantity'] } }
        }},
        { $lookup: {
            from: 'products',
            localField: '_id.productId',
            foreignField: '_id',
            as: 'product'
        }},
        { $unwind: '$product' },
        { $project: {
            _id: 0,
            productName: '$product.name',
            month: '$_id.month',
            year: '$_id.year',
            totalQuantity: 1,
            totalRevenue: 1
        }}
    ]
);

// View для публичного API (скрытие чувствительных данных)
db.createView(
    'publicUsers',
    'users',
    [
        { $match: { isPublic: true } },
        { $project: {
            username: 1,
            avatar: 1,
            bio: 1,
            createdAt: 1
            // пароль, email и другие приватные поля исключены
        }}
    ]
);
```

### Ограничения Views

- Только для чтения (нельзя insert/update/delete)
- Нельзя создавать индексы на view
- Некоторые операции не поддерживаются ($out, $merge)
- Производительность зависит от сложности pipeline

---

## 9. Time Series Collections

Time Series Collections оптимизированы для хранения временных рядов данных с автоматическим сжатием и эффективными запросами по времени.

### Создание Time Series Collection

```javascript
db.createCollection('sensorData', {
    timeseries: {
        timeField: 'timestamp',         // обязательное поле времени
        metaField: 'sensorId',          // поле метаданных (опционально)
        granularity: 'seconds'          // seconds | minutes | hours
    },
    expireAfterSeconds: 86400 * 30      // TTL — 30 дней (опционально)
});

// Пример с подробными настройками
db.createCollection('metrics', {
    timeseries: {
        timeField: 'ts',
        metaField: 'metadata',
        granularity: 'minutes',
        bucketMaxSpanSeconds: 3600,     // максимальный интервал bucket (1 час)
        bucketRoundingSeconds: 60       // округление bucket
    }
});
```

### Вставка данных

```javascript
// Вставка показаний датчиков
db.sensorData.insertMany([
    {
        sensorId: 'sensor-001',
        timestamp: new Date(),
        temperature: 23.5,
        humidity: 45,
        pressure: 1013.25
    },
    {
        sensorId: 'sensor-002',
        timestamp: new Date(),
        temperature: 21.0,
        humidity: 50,
        pressure: 1012.80
    }
]);

// Использование metadata как объекта
db.metrics.insertOne({
    ts: new Date(),
    metadata: {
        region: 'us-east',
        service: 'api',
        host: 'server-01'
    },
    cpu: 45.2,
    memory: 2048,
    requests: 1523
});
```

### Запросы к Time Series

```javascript
// Фильтрация по времени
db.sensorData.find({
    timestamp: {
        $gte: new Date('2024-01-01'),
        $lt: new Date('2024-01-02')
    }
});

// Агрегация по интервалам
db.sensorData.aggregate([
    { $match: { sensorId: 'sensor-001' } },
    { $group: {
        _id: {
            $dateTrunc: {
                date: '$timestamp',
                unit: 'hour'
            }
        },
        avgTemperature: { $avg: '$temperature' },
        maxTemperature: { $max: '$temperature' },
        minTemperature: { $min: '$temperature' },
        count: { $sum: 1 }
    }},
    { $sort: { _id: 1 } }
]);

// Использование $setWindowFields для скользящего среднего
db.sensorData.aggregate([
    { $match: { sensorId: 'sensor-001' } },
    { $setWindowFields: {
        partitionBy: '$sensorId',
        sortBy: { timestamp: 1 },
        output: {
            movingAvgTemp: {
                $avg: '$temperature',
                window: { documents: [-5, 0] }  // последние 6 документов
            }
        }
    }}
]);
```

### Вторичные индексы на Time Series

```javascript
// Индекс на metaField (автоматически создаётся составной индекс с timeField)
db.sensorData.createIndex({ sensorId: 1, timestamp: 1 });

// Индекс на поля измерений
db.sensorData.createIndex({ temperature: 1 });
```

### Практический пример: мониторинг серверов

```javascript
// Создание коллекции для метрик серверов
db.createCollection('serverMetrics', {
    timeseries: {
        timeField: 'timestamp',
        metaField: 'server',
        granularity: 'seconds'
    },
    expireAfterSeconds: 604800  // хранить 7 дней
});

// Вставка метрик
function recordMetrics(serverId, metrics) {
    db.serverMetrics.insertOne({
        timestamp: new Date(),
        server: { id: serverId, region: 'eu-west' },
        cpu: metrics.cpu,
        memory: metrics.memory,
        diskIO: metrics.diskIO,
        networkIn: metrics.networkIn,
        networkOut: metrics.networkOut
    });
}

// Аналитика: среднее использование CPU по серверам за последний час
db.serverMetrics.aggregate([
    { $match: {
        timestamp: { $gte: new Date(Date.now() - 3600000) }
    }},
    { $group: {
        _id: '$server.id',
        avgCPU: { $avg: '$cpu' },
        maxCPU: { $max: '$cpu' },
        dataPoints: { $sum: 1 }
    }},
    { $sort: { avgCPU: -1 } }
]);
```

---

## 10. Дополнительные полезные концепции

### Bulk Operations (Массовые операции)

```javascript
// Ordered bulk (остановится при первой ошибке)
const bulk = db.products.initializeOrderedBulkOp();
bulk.insert({ name: 'Product 1', price: 100 });
bulk.insert({ name: 'Product 2', price: 200 });
bulk.find({ price: { $lt: 50 } }).update({ $set: { onSale: true } });
bulk.find({ stock: 0 }).delete();
bulk.execute();

// Unordered bulk (выполнит все операции, соберёт ошибки)
const unorderedBulk = db.products.initializeUnorderedBulkOp();
// ... операции
unorderedBulk.execute();

// Modern bulkWrite
db.products.bulkWrite([
    { insertOne: { document: { name: 'New Product', price: 150 } } },
    { updateMany: {
        filter: { category: 'electronics' },
        update: { $inc: { price: 10 } }
    }},
    { deleteOne: { filter: { discontinued: true } } }
], { ordered: false });
```

### Collation (Сортировка с учётом локали)

```javascript
// Создание коллекции с collation
db.createCollection('users', {
    collation: { locale: 'ru', strength: 2 }
});

// Запрос с collation
db.users.find({ name: 'Алексей' }).collation({
    locale: 'ru',
    strength: 1  // регистронезависимый поиск
});

// Индекс с collation
db.users.createIndex(
    { name: 1 },
    { collation: { locale: 'ru' } }
);

// Сортировка с учётом русского алфавита
db.users.find().sort({ name: 1 }).collation({ locale: 'ru' });
```

### Read/Write Concern

```javascript
// Write Concern — гарантии записи
db.orders.insertOne(
    { item: 'laptop', quantity: 1 },
    { writeConcern: {
        w: 'majority',      // подтверждение от большинства узлов
        j: true,            // запись в journal
        wtimeout: 5000      // таймаут 5 секунд
    }}
);

// Read Concern — консистентность чтения
db.orders.find().readConcern('majority');  // только закоммиченные данные

// Read Preference — выбор узла для чтения
db.orders.find().readPref('secondaryPreferred');  // предпочитать secondary
```

### Sessions и Transactions (ACID транзакции)

```javascript
const session = db.getMongo().startSession();

session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' }
});

try {
    const accounts = session.getDatabase('bank').accounts;

    // Перевод денег между счетами
    accounts.updateOne(
        { _id: 'account1' },
        { $inc: { balance: -100 } },
        { session }
    );

    accounts.updateOne(
        { _id: 'account2' },
        { $inc: { balance: 100 } },
        { session }
    );

    session.commitTransaction();
} catch (error) {
    session.abortTransaction();
    throw error;
} finally {
    session.endSession();
}
```

---

## Сводная таблица концепций

| Концепция | Применение | Ключевые особенности |
|-----------|------------|---------------------|
| Change Streams | Real-time синхронизация | Требует replica set |
| GridFS | Файлы > 16 МБ | Chunks по 255 КБ |
| Capped Collections | Логи, очереди | FIFO, фикс. размер |
| TTL индексы | Временные данные | Авто-удаление |
| Text Search | Полнотекстовый поиск | Стемминг, веса |
| Geospatial | Геоданные | 2dsphere индекс |
| Schema Validation | Валидация данных | JSON Schema |
| Views | Виртуальные коллекции | Только чтение |
| Time Series | Временные ряды | Сжатие, buckets |

---

## Полезные ресурсы

- [MongoDB Manual — Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)
- [MongoDB Manual — GridFS](https://www.mongodb.com/docs/manual/core/gridfs/)
- [MongoDB Manual — Time Series](https://www.mongodb.com/docs/manual/core/timeseries-collections/)
- [MongoDB Manual — Schema Validation](https://www.mongodb.com/docs/manual/core/schema-validation/)
- [MongoDB Manual — Geospatial Queries](https://www.mongodb.com/docs/manual/geospatial-queries/)
