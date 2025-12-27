# Developer Tools

## Введение

MongoDB предоставляет богатую экосистему инструментов для разработчиков, которая охватывает все аспекты работы с базой данных: от визуального исследования данных до автоматизации DevOps-процессов. Эти инструменты помогают эффективно разрабатывать, отлаживать, мониторить и оптимизировать приложения, использующие MongoDB.

Основные категории инструментов:
- **GUI-инструменты** — визуальные приложения для работы с данными
- **CLI-инструменты** — командная строка для администрирования и автоматизации
- **Драйверы** — библиотеки для интеграции с языками программирования
- **Инструменты мониторинга** — утилиты для анализа производительности

---

## MongoDB Compass

### Обзор

MongoDB Compass — это официальное GUI-приложение для работы с MongoDB. Это мощный инструмент для визуального исследования данных, построения запросов и анализа схемы.

### Установка и настройка

**Установка:**
```bash
# macOS (через Homebrew)
brew install --cask mongodb-compass

# Ubuntu/Debian
wget https://downloads.mongodb.com/compass/mongodb-compass_1.40.4_amd64.deb
sudo dpkg -i mongodb-compass_1.40.4_amd64.deb

# Windows — скачать установщик с официального сайта
# https://www.mongodb.com/try/download/compass
```

**Подключение к базе данных:**
```
# Connection String формат
mongodb://username:password@host:port/database

# Примеры
mongodb://localhost:27017
mongodb+srv://user:pass@cluster.mongodb.net/mydb

# С опциями
mongodb://localhost:27017/?authSource=admin&readPreference=primary
```

### Основные функции

**1. Визуализация данных**
- Просмотр коллекций в табличном и JSON-форматах
- Редактирование документов прямо в интерфейсе
- Вставка новых документов
- Удаление и обновление записей

**2. Построение запросов**
```javascript
// Визуальный конструктор генерирует запросы
// Filter
{ status: "active", age: { $gte: 18 } }

// Project
{ name: 1, email: 1, _id: 0 }

// Sort
{ createdAt: -1 }
```

**3. Управление индексами**
- Просмотр существующих индексов
- Создание новых индексов через GUI
- Анализ использования индексов
- Удаление неиспользуемых индексов

### Aggregation Pipeline Builder

Compass предоставляет визуальный конструктор aggregation pipelines — один из самых полезных инструментов для сложных запросов.

```javascript
// Пример pipeline, созданного в конструкторе
[
  // Stage 1: Фильтрация
  {
    $match: {
      status: "completed",
      createdAt: { $gte: ISODate("2024-01-01") }
    }
  },
  // Stage 2: Группировка
  {
    $group: {
      _id: "$category",
      totalSales: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  // Stage 3: Сортировка
  {
    $sort: { totalSales: -1 }
  },
  // Stage 4: Лимит
  {
    $limit: 10
  }
]
```

**Возможности конструктора:**
- Добавление стадий перетаскиванием
- Предпросмотр результатов на каждой стадии
- Автодополнение полей и операторов
- Экспорт pipeline в код на разных языках

### Schema Analysis

Compass автоматически анализирует структуру документов в коллекции:

- **Типы полей** — показывает распределение типов данных
- **Частота** — процент документов, содержащих поле
- **Значения** — визуализация распределения значений
- **Обнаружение аномалий** — нетипичные значения и типы

Это особенно полезно для:
- Понимания структуры унаследованных данных
- Обнаружения несогласованности в схеме
- Планирования индексов
- Валидации данных

---

## mongosh (MongoDB Shell)

### Обзор

`mongosh` (MongoDB Shell) — это современная интерактивная оболочка JavaScript для работы с MongoDB. Она заменила устаревший `mongo` shell и предоставляет улучшенный опыт работы.

### Установка

```bash
# macOS
brew install mongosh

# Ubuntu/Debian
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update
sudo apt-get install -y mongodb-mongosh

# npm (кроссплатформенно)
npm install -g mongosh


# Windows — через MSI установщик или npm
```

### Подключение

```bash
# Локальное подключение
mongosh

# К конкретному хосту и БД
mongosh "mongodb://localhost:27017/mydb"

# К MongoDB Atlas
mongosh "mongodb+srv://cluster.mongodb.net/mydb" --apiVersion 1 --username admin

# С аутентификацией
mongosh --host localhost --port 27017 -u admin -p password --authenticationDatabase admin
```

### Основные команды

```javascript
// Навигация
show dbs                    // Список баз данных
use mydb                    // Переключиться на БД
show collections            // Список коллекций
db                          // Текущая база данных

// CRUD операции
db.users.insertOne({ name: "Ivan", age: 25 })
db.users.find({ age: { $gte: 18 } })
db.users.updateOne({ name: "Ivan" }, { $set: { age: 26 } })
db.users.deleteOne({ name: "Ivan" })

// Информация
db.users.countDocuments()
db.users.estimatedDocumentCount()
db.users.getIndexes()
db.stats()

// Администрирование
db.createCollection("logs", { capped: true, size: 1000000 })
db.users.createIndex({ email: 1 }, { unique: true })
db.users.drop()

// Помощь
help                        // Общая справка
db.help()                   // Справка по командам БД
db.users.help()             // Справка по коллекции
```

### Скрипты и автоматизация

**Выполнение скрипта:**
```bash
# Запуск JavaScript файла
mongosh "mongodb://localhost:27017/mydb" script.js

# С передачей переменных
mongosh --eval "var collection='users'" script.js

# Выполнение команды напрямую
mongosh --eval "db.users.countDocuments()"
```

**Пример скрипта (migration.js):**
```javascript
// migration.js - миграция данных
const db = db.getSiblingDB('production');

// Добавляем новое поле ко всем документам
const result = db.users.updateMany(
  { status: { $exists: false } },
  { $set: { status: 'active', updatedAt: new Date() } }
);

print(`Modified ${result.modifiedCount} documents`);

// Создаем индекс
db.users.createIndex(
  { email: 1, status: 1 },
  { name: 'email_status_idx' }
);

print('Migration completed successfully');
```

**Интерактивные функции:**
```javascript
// Определение собственных функций
function findActiveUsers(limit = 10) {
  return db.users.find({ status: 'active' }).limit(limit).toArray();
}

// Использование
findActiveUsers(5);

// Форматирование вывода
db.users.find().forEach(user => {
  print(`User: ${user.name}, Email: ${user.email}`);
});
```

### Отличия от legacy mongo shell

| Особенность | legacy mongo | mongosh |
|-------------|--------------|---------|
| Движок JS | SpiderMonkey | Node.js |
| Синтаксис | ES5 | ES2020+ |
| Подсветка синтаксиса | Нет | Да |
| Автодополнение | Базовое | Продвинутое |
| Async/await | Нет | Да |
| Вывод | Простой | Цветной, форматированный |
| Расширяемость | Ограниченная | npm пакеты |

```javascript
// В mongosh можно использовать современный JavaScript
const users = await db.users.find({ active: true }).toArray();
const names = users.map(u => u.name);
console.log(names);

// Деструктуризация
const { insertedId } = await db.users.insertOne({ name: "Test" });
```

---

## MongoDB CLI for Cloud (Atlas CLI)

### Обзор

Atlas CLI — инструмент командной строки для управления MongoDB Atlas (облачным сервисом MongoDB). Позволяет автоматизировать DevOps-процессы и управлять инфраструктурой из терминала.

### Установка

```bash
# macOS
brew install mongodb-atlas-cli

# Linux
curl -sL https://raw.githubusercontent.com/mongodb/mongodb-atlas-cli/master/install.sh | bash

# npm
npm install -g mongodb-atlas-cli
```

### Аутентификация

```bash
# Интерактивная настройка
atlas auth login

# Через API ключи (для CI/CD)
atlas config init
# Затем ввести Public Key и Private Key

# Проверка конфигурации
atlas config list
```

### Управление кластерами

```bash
# Список кластеров
atlas clusters list

# Создание кластера
atlas clusters create myCluster \
  --provider AWS \
  --region US_EAST_1 \
  --tier M10 \
  --members 3

# Информация о кластере
atlas clusters describe myCluster

# Получение connection string
atlas clusters connectionStrings describe myCluster

# Пауза кластера (экономия средств)
atlas clusters pause myCluster

# Возобновление
atlas clusters start myCluster

# Удаление
atlas clusters delete myCluster
```

### Управление пользователями и доступом

```bash
# Создание пользователя БД
atlas dbusers create \
  --username myuser \
  --password mypassword \
  --role readWriteAnyDatabase

# Список пользователей
atlas dbusers list

# Настройка IP Whitelist
atlas accessLists create --entry "192.168.1.0/24" --comment "Office network"

# Разрешить доступ отовсюду (не рекомендуется для production)
atlas accessLists create --entry "0.0.0.0/0"
```

### Автоматизация DevOps

**Пример скрипта для CI/CD:**
```bash
#!/bin/bash
# deploy-cluster.sh

set -e

PROJECT_ID="your-project-id"
CLUSTER_NAME="staging-cluster"

# Проверка существования кластера
if atlas clusters describe $CLUSTER_NAME --projectId $PROJECT_ID > /dev/null 2>&1; then
  echo "Cluster exists, updating..."
  atlas clusters update $CLUSTER_NAME \
    --projectId $PROJECT_ID \
    --tier M20
else
  echo "Creating new cluster..."
  atlas clusters create $CLUSTER_NAME \
    --projectId $PROJECT_ID \
    --provider AWS \
    --region US_EAST_1 \
    --tier M10
fi

# Ожидание готовности
atlas clusters watch $CLUSTER_NAME --projectId $PROJECT_ID

echo "Cluster is ready!"
```

**Экспорт/импорт данных:**
```bash
# Экспорт базы данных
atlas clusters sampleData load myCluster --sampleDatasetName sample_mflix

# Резервное копирование (Snapshots)
atlas backups snapshots list myCluster
atlas backups snapshots create myCluster --description "Pre-release backup"

# Восстановление
atlas backups restores start myCluster \
  --snapshotId <snapshot-id> \
  --targetClusterName myCluster-restored
```

---

## VS Code Extension

### Обзор

MongoDB for VS Code — официальное расширение для Visual Studio Code, которое позволяет работать с MongoDB прямо из редактора кода.

### Установка

1. Открыть VS Code
2. Extensions (Ctrl+Shift+X)
3. Поиск "MongoDB for VS Code"
4. Install

### Возможности

**1. Подключение к базам данных**
- Поддержка connection strings
- Сохранение нескольких подключений
- Интеграция с MongoDB Atlas

**2. Навигация по данным**
- Просмотр баз данных и коллекций в боковой панели
- Просмотр документов
- Просмотр схемы коллекции

**3. Операции с данными**
- Создание, редактирование, удаление документов
- Выполнение запросов
- Создание индексов

### Работа с Playgrounds

MongoDB Playgrounds — это интерактивная среда для написания и выполнения запросов MongoDB прямо в VS Code.

**Создание Playground:**
```javascript
// playground-1.mongodb.js

// Выбор базы данных
use('ecommerce');

// Создание коллекции и вставка данных
db.products.insertMany([
  { name: 'Laptop', price: 999, category: 'Electronics', inStock: true },
  { name: 'Headphones', price: 199, category: 'Electronics', inStock: true },
  { name: 'Coffee Mug', price: 15, category: 'Kitchen', inStock: false }
]);

// Запрос с фильтрацией
db.products.find({ category: 'Electronics', price: { $lt: 500 } });

// Aggregation pipeline
db.products.aggregate([
  { $match: { inStock: true } },
  { $group: { _id: '$category', avgPrice: { $avg: '$price' } } },
  { $sort: { avgPrice: -1 } }
]);

// Обновление
db.products.updateMany(
  { category: 'Electronics' },
  { $mul: { price: 0.9 } }  // Скидка 10%
);
```

**Особенности Playgrounds:**
- Автодополнение команд и полей
- Подсветка синтаксиса MongoDB
- Выполнение всего файла или выделенного фрагмента
- Просмотр результатов в панели вывода
- Экспорт в разные языки программирования

**Полезные сниппеты:**
```javascript
// Шаблон для тестирования производительности
const start = Date.now();
const result = db.collection.find({ ... }).toArray();
const elapsed = Date.now() - start;
print(`Query executed in ${elapsed}ms, returned ${result.length} documents`);

// Explain для анализа запроса
db.products.find({ category: 'Electronics' }).explain('executionStats');
```

---

## Драйверы для языков программирования

### Node.js

**Native MongoDB Driver:**
```bash
npm install mongodb
```

```javascript
const { MongoClient } = require('mongodb');

async function main() {
  const uri = 'mongodb://localhost:27017';
  const client = new MongoClient(uri);

  try {
    await client.connect();
    const db = client.db('myapp');
    const users = db.collection('users');

    // Вставка
    const result = await users.insertOne({
      name: 'Ivan',
      email: 'ivan@example.com',
      createdAt: new Date()
    });
    console.log(`Inserted with _id: ${result.insertedId}`);

    // Поиск
    const user = await users.findOne({ email: 'ivan@example.com' });
    console.log(user);

    // Обновление
    await users.updateOne(
      { _id: result.insertedId },
      { $set: { verified: true } }
    );

    // Aggregation
    const stats = await users.aggregate([
      { $group: { _id: null, count: { $sum: 1 } } }
    ]).toArray();

  } finally {
    await client.close();
  }
}

main().catch(console.error);
```

**Mongoose (ODM):**
```bash
npm install mongoose
```

```javascript
const mongoose = require('mongoose');

// Подключение
mongoose.connect('mongodb://localhost:27017/myapp');

// Определение схемы
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: { type: Number, min: 0 },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  createdAt: { type: Date, default: Date.now }
});

// Middleware (hooks)
userSchema.pre('save', function(next) {
  this.email = this.email.toLowerCase();
  next();
});

// Виртуальные поля
userSchema.virtual('isAdult').get(function() {
  return this.age >= 18;
});

// Методы экземпляра
userSchema.methods.getPublicProfile = function() {
  return { name: this.name, role: this.role };
};

// Статические методы
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

const User = mongoose.model('User', userSchema);

// Использование
async function example() {
  const user = new User({ name: 'Ivan', email: 'Ivan@Example.com', age: 25 });
  await user.save();

  const found = await User.findByEmail('ivan@example.com');
  console.log(found.getPublicProfile());
}
```

### Python

**PyMongo (синхронный):**
```bash
pip install pymongo
```

```python
from pymongo import MongoClient
from datetime import datetime

# Подключение
client = MongoClient('mongodb://localhost:27017/')
db = client['myapp']
users = db['users']

# Вставка
result = users.insert_one({
    'name': 'Ivan',
    'email': 'ivan@example.com',
    'created_at': datetime.utcnow()
})
print(f'Inserted with _id: {result.inserted_id}')

# Поиск
user = users.find_one({'email': 'ivan@example.com'})
print(user)

# Поиск нескольких документов
for user in users.find({'age': {'$gte': 18}}).limit(10):
    print(user['name'])

# Обновление
users.update_one(
    {'email': 'ivan@example.com'},
    {'$set': {'verified': True}}
)

# Aggregation
pipeline = [
    {'$match': {'verified': True}},
    {'$group': {'_id': '$role', 'count': {'$sum': 1}}}
]
for doc in users.aggregate(pipeline):
    print(doc)

# Индексы
users.create_index('email', unique=True)
users.create_index([('name', 1), ('created_at', -1)])
```

**Motor (асинхронный):**
```bash
pip install motor
```

```python
import asyncio
from motor.motor_asyncio import AsyncIOMotorClient

async def main():
    client = AsyncIOMotorClient('mongodb://localhost:27017/')
    db = client['myapp']
    users = db['users']

    # Асинхронные операции
    result = await users.insert_one({'name': 'Ivan', 'age': 25})
    print(f'Inserted: {result.inserted_id}')

    user = await users.find_one({'name': 'Ivan'})
    print(user)

    # Асинхронный курсор
    async for user in users.find({'age': {'$gte': 18}}):
        print(user['name'])

asyncio.run(main())
```

### Java

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver-sync</artifactId>
    <version>4.11.0</version>
</dependency>
```

```java
import com.mongodb.client.*;
import org.bson.Document;
import java.util.Arrays;

public class MongoDBExample {
    public static void main(String[] args) {
        try (MongoClient client = MongoClients.create("mongodb://localhost:27017")) {
            MongoDatabase db = client.getDatabase("myapp");
            MongoCollection<Document> users = db.getCollection("users");

            // Вставка
            Document user = new Document("name", "Ivan")
                    .append("email", "ivan@example.com")
                    .append("age", 25);
            users.insertOne(user);

            // Поиск
            Document found = users.find(new Document("name", "Ivan")).first();
            System.out.println(found.toJson());

            // Aggregation
            users.aggregate(Arrays.asList(
                new Document("$match", new Document("age", new Document("$gte", 18))),
                new Document("$group", new Document("_id", "$role")
                        .append("count", new Document("$sum", 1)))
            )).forEach(doc -> System.out.println(doc.toJson()));
        }
    }
}
```

### Go

```bash
go get go.mongodb.org/mongo-driver/mongo
```

```go
package main

import (
    "context"
    "fmt"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type User struct {
    Name      string    `bson:"name"`
    Email     string    `bson:"email"`
    Age       int       `bson:"age"`
    CreatedAt time.Time `bson:"created_at"`
}

func main() {
    ctx := context.Background()

    client, err := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        panic(err)
    }
    defer client.Disconnect(ctx)

    users := client.Database("myapp").Collection("users")

    // Вставка
    user := User{Name: "Ivan", Email: "ivan@example.com", Age: 25, CreatedAt: time.Now()}
    result, err := users.InsertOne(ctx, user)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Inserted: %v\n", result.InsertedID)

    // Поиск
    var found User
    err = users.FindOne(ctx, bson.M{"name": "Ivan"}).Decode(&found)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found: %+v\n", found)

    // Поиск нескольких
    cursor, err := users.Find(ctx, bson.M{"age": bson.M{"$gte": 18}})
    if err != nil {
        panic(err)
    }
    defer cursor.Close(ctx)

    for cursor.Next(ctx) {
        var u User
        cursor.Decode(&u)
        fmt.Println(u.Name)
    }
}
```

### Выбор драйвера

| Критерий | Native Driver | ODM/ORM |
|----------|--------------|---------|
| Производительность | Максимальная | Небольшой overhead |
| Гибкость | Полная | Ограничена схемой |
| Валидация | Ручная | Встроенная |
| Миграции | Нет | Часто поддерживаются |
| Кривая обучения | Нужно знать MongoDB API | Абстракция от деталей |
| Типизация | Минимальная | Полная (с моделями) |

**Рекомендации:**
- **Native Driver** — для микросервисов, высоконагруженных систем, когда нужен полный контроль
- **ODM (Mongoose, Motor)** — для типичных CRUD-приложений, когда важна валидация и структура

---

## Инструменты мониторинга и профилирования

### mongostat

Утилита для мониторинга состояния сервера MongoDB в реальном времени.

```bash
# Базовый запуск (обновление каждую секунду)
mongostat

# С аутентификацией
mongostat --host localhost --port 27017 -u admin -p password --authenticationDatabase admin

# Вывод N строк
mongostat --rowcount 10

# Интервал обновления (миллисекунды)
mongostat 2000

# JSON формат
mongostat --json
```

**Основные метрики:**
```
insert  query update delete getmore command dirty used flushes vsize  res qrw arw net_in net_out conn
  *0     *0     *0     *0       0     2|0  0.0% 0.0%       0  1.5G  80M 0|0 1|0   158b   59.8k    2
```

- `insert/query/update/delete` — операции в секунду
- `dirty` — процент грязных страниц в кеше
- `used` — использование кеша WiredTiger
- `vsize/res` — виртуальная и резидентная память
- `qrw/arw` — очереди чтения/записи
- `conn` — количество подключений

### mongotop

Показывает время, затраченное на операции чтения и записи по коллекциям.

```bash
# Базовый запуск
mongotop

# Интервал 5 секунд
mongotop 5

# Вывод N строк
mongotop --rowcount 10

# JSON формат
mongotop --json
```

**Пример вывода:**
```
                    ns    total    read    write
    myapp.users           10ms     8ms      2ms
    myapp.orders           5ms     3ms      2ms
    myapp.products         2ms     2ms      0ms
```

### Database Profiler

Встроенный профилировщик для детального анализа запросов.

**Настройка профилировщика:**
```javascript
// В mongosh

// Проверка текущего уровня
db.getProfilingStatus()

// Уровни профилирования:
// 0 - выключен
// 1 - профилировать медленные запросы (> slowms)
// 2 - профилировать все запросы

// Включить для медленных запросов (>100ms)
db.setProfilingLevel(1, { slowms: 100 })

// Включить для всех запросов
db.setProfilingLevel(2)

// Выключить
db.setProfilingLevel(0)
```

**Анализ профилей:**
```javascript
// Коллекция system.profile хранит данные профилирования

// Последние 10 медленных запросов
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// Запросы дольше 100ms
db.system.profile.find({ millis: { $gt: 100 } })

// Запросы без использования индекса (COLLSCAN)
db.system.profile.find({
  'planSummary': 'COLLSCAN'
}).sort({ millis: -1 })

// Группировка по типам операций
db.system.profile.aggregate([
  { $group: {
    _id: '$op',
    count: { $sum: 1 },
    avgTime: { $avg: '$millis' }
  }}
])

// Самые частые медленные запросы
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  { $group: {
    _id: '$command.filter',
    count: { $sum: 1 },
    avgMillis: { $avg: '$millis' }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

**Пример профиля запроса:**
```javascript
{
  "op": "query",
  "ns": "myapp.users",
  "command": {
    "find": "users",
    "filter": { "email": "ivan@example.com" }
  },
  "keysExamined": 1,      // Просмотрено записей индекса
  "docsExamined": 1,      // Просмотрено документов
  "nreturned": 1,         // Возвращено документов
  "millis": 0,            // Время выполнения
  "planSummary": "IXSCAN { email: 1 }",  // Использованный план
  "ts": ISODate("2024-01-15T10:30:00Z")
}
```

### MongoDB Atlas Monitoring

MongoDB Atlas предоставляет комплексный мониторинг в облаке:

**Доступные метрики:**
- **Операции** — queries, inserts, updates, deletes в секунду
- **Производительность** — latency, opcounters, scan/document ratio
- **Память** — использование RAM, page faults
- **Сеть** — входящий/исходящий трафик, количество подключений
- **Диск** — IOPS, throughput, disk utilization
- **Репликация** — replication lag, oplog window

**Настройка алертов:**
```bash
# Через Atlas CLI
atlas alerts configs create \
  --event OUTSIDE_METRIC_THRESHOLD \
  --metricName QUERY_TARGETING_SCANNED_OBJECTS_PER_RETURNED \
  --operator GREATER_THAN \
  --threshold 1000 \
  --notificationType EMAIL \
  --notificationEmailAddress admin@example.com
```

**Real-Time Performance Panel:**
- Визуализация активных операций в реальном времени
- Hottest collections — наиболее нагруженные коллекции
- Slowest operations — самые медленные запросы
- Возможность kill операций прямо из интерфейса

**Query Profiler в Atlas:**
```javascript
// Доступ через Atlas UI: Cluster -> Performance Advisor

// Показывает:
// - Рекомендации по индексам
// - Медленные запросы
// - Неоптимальные паттерны доступа
// - Предложения по оптимизации схемы
```

### Сводка инструментов мониторинга

| Инструмент | Тип | Использование |
|------------|-----|---------------|
| mongostat | CLI | Общее состояние сервера в реальном времени |
| mongotop | CLI | Анализ нагрузки по коллекциям |
| Database Profiler | Встроенный | Детальный анализ запросов |
| Atlas Monitoring | Cloud | Комплексный мониторинг и алерты |
| Compass | GUI | Визуальный анализ производительности |

---

## Заключение

Экосистема инструментов MongoDB покрывает все аспекты разработки и эксплуатации:

1. **MongoDB Compass** — для визуального исследования данных и построения запросов
2. **mongosh** — для скриптов, автоматизации и администрирования
3. **Atlas CLI** — для управления облачной инфраструктурой
4. **VS Code Extension** — для интеграции с процессом разработки
5. **Драйверы** — для интеграции с приложениями на любом языке
6. **Инструменты мониторинга** — для анализа и оптимизации производительности

Правильное использование этих инструментов значительно повышает продуктивность разработки и помогает создавать надежные, производительные приложения.
