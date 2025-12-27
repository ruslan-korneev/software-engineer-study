# Основы MongoDB

## Что такое MongoDB

**MongoDB** — это документо-ориентированная NoSQL база данных с открытым исходным кодом. Название происходит от слова "humongous" (огромный), что отражает её способность работать с большими объёмами данных.

### Ключевые характеристики

- **Document-oriented**: данные хранятся в виде документов (JSON-подобный формат BSON)
- **Schema-flexible**: документы в одной коллекции могут иметь разную структуру
- **Horizontally scalable**: поддержка шардирования для распределения данных
- **High availability**: встроенная репликация через replica sets

### История

- 2007 — разработка началась в компании 10gen (сейчас MongoDB Inc.)
- 2009 — первый публичный релиз как open-source
- 2013 — MongoDB 2.4 с улучшенной геопространственной поддержкой
- 2018 — MongoDB 4.0 с поддержкой ACID-транзакций
- 2024 — MongoDB 7.x с векторным поиском и улучшенной производительностью

---

## Document-Oriented подход vs Реляционные БД

### Реляционные базы данных (SQL)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   users     │     │   orders    │     │  products   │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ id          │←────│ user_id     │     │ id          │
│ name        │     │ id          │────→│ name        │
│ email       │     │ product_id  │     │ price       │
└─────────────┘     └─────────────┘     └─────────────┘
```

- Данные нормализованы и разбиты по таблицам
- JOIN для объединения данных
- Строгая схема (ALTER TABLE для изменений)

### MongoDB (Document Store)

```json
{
  "_id": ObjectId("..."),
  "name": "Иван Петров",
  "email": "ivan@example.com",
  "orders": [
    {
      "date": ISODate("2024-01-15"),
      "products": [
        { "name": "Ноутбук", "price": 85000 },
        { "name": "Мышь", "price": 2500 }
      ],
      "total": 87500
    }
  ]
}
```

- Данные денормализованы и хранятся вместе
- Нет JOIN — данные читаются одним запросом
- Гибкая схема — документы могут отличаться

### Сравнение терминологии

| SQL              | MongoDB          |
|------------------|------------------|
| Database         | Database         |
| Table            | Collection       |
| Row              | Document         |
| Column           | Field            |
| Primary Key      | _id              |
| JOIN             | $lookup / embed  |
| Index            | Index            |

---

## Архитектура MongoDB

### Основные компоненты

```
┌─────────────────────────────────────────────────────────┐
│                      Приложение                          │
│                    (MongoDB Driver)                      │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   mongos (Router)                        │
│              (только для sharded cluster)               │
└──────────────────────────┬──────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  Shard 1 │    │  Shard 2 │    │  Shard 3 │
    │ (RS)     │    │ (RS)     │    │ (RS)     │
    └──────────┘    └──────────┘    └──────────┘
```

### mongod — основной процесс

**mongod** — это демон базы данных, который обрабатывает запросы, управляет данными и выполняет фоновые операции.

```bash
# Запуск с конфигурацией
mongod --config /etc/mongod.conf

# Основные параметры
mongod --dbpath /data/db --port 27017 --bind_ip 0.0.0.0
```

### Replica Set — отказоустойчивость

Replica Set — это группа mongod-процессов, поддерживающих одинаковый набор данных.

```
┌─────────────┐
│   Primary   │  ← Все записи
│   (mongod)  │
└──────┬──────┘
       │ репликация
       ├────────────────┐
       ▼                ▼
┌─────────────┐  ┌─────────────┐
│  Secondary  │  │  Secondary  │  ← Чтение (опционально)
│   (mongod)  │  │   (mongod)  │
└─────────────┘  └─────────────┘
```

- **Primary**: принимает все операции записи
- **Secondary**: реплицирует данные с Primary, может обслуживать чтение
- **Arbiter**: участвует в выборах, но не хранит данные

### mongos — роутер для шардинга

**mongos** — маршрутизатор запросов в sharded cluster.

```bash
mongos --configdb configReplSet/cfg1:27019,cfg2:27019,cfg3:27019
```

---

## Установка MongoDB

### macOS (Homebrew)

```bash
# Добавить tap MongoDB
brew tap mongodb/brew

# Установить Community Edition
brew install mongodb-community@7.0

# Запустить как сервис
brew services start mongodb-community@7.0

# Или запустить вручную
mongod --config /opt/homebrew/etc/mongod.conf
```

### Ubuntu/Debian

```bash
# Импорт GPG ключа
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Добавление репозитория
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
   https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Установка
sudo apt-get update
sudo apt-get install -y mongodb-org

# Запуск
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Docker

```bash
# Запуск контейнера
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:7

# Подключение
docker exec -it mongodb mongosh -u admin -p secret
```

---

## MongoDB Shell (mongosh)

**mongosh** — современная интерактивная оболочка для работы с MongoDB (замена устаревшего `mongo`).

### Подключение

```bash
# Локальное подключение
mongosh

# С указанием хоста и порта
mongosh "mongodb://localhost:27017"

# С аутентификацией
mongosh "mongodb://user:password@localhost:27017/mydb"

# К replica set
mongosh "mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0"

# К MongoDB Atlas
mongosh "mongodb+srv://cluster.xxxxx.mongodb.net/mydb" --apiVersion 1 --username user
```

### Базовые команды

```javascript
// Показать все базы данных
show dbs

// Переключиться на базу данных (создаст при первой записи)
use myDatabase

// Показать текущую базу данных
db

// Показать все коллекции
show collections

// Справка
help

// Выход
exit
```

### Работа с данными

```javascript
// Вставка документа
db.users.insertOne({
  name: "Алексей",
  email: "alex@example.com",
  age: 28
})

// Вставка нескольких документов
db.users.insertMany([
  { name: "Мария", age: 25 },
  { name: "Дмитрий", age: 32 }
])

// Поиск всех документов
db.users.find()

// Поиск с фильтром
db.users.find({ age: { $gte: 25 } })

// Поиск одного документа
db.users.findOne({ name: "Алексей" })

// Обновление
db.users.updateOne(
  { name: "Алексей" },
  { $set: { age: 29 } }
)

// Удаление
db.users.deleteOne({ name: "Мария" })

// Подсчёт документов
db.users.countDocuments({ age: { $gte: 25 } })
```

### Полезные методы

```javascript
// Форматированный вывод
db.users.find().pretty()

// Ограничение количества
db.users.find().limit(10)

// Пропуск документов
db.users.find().skip(5)

// Сортировка
db.users.find().sort({ age: -1 })

// Проекция (выбор полей)
db.users.find({}, { name: 1, email: 1, _id: 0 })

// Объяснение плана запроса
db.users.find({ age: 25 }).explain("executionStats")
```

---

## Подключение из приложений

### Node.js (официальный драйвер)

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://localhost:27017";
const client = new MongoClient(uri);

async function run() {
  try {
    await client.connect();
    const database = client.db('myapp');
    const users = database.collection('users');

    // Вставка
    const result = await users.insertOne({
      name: "Ольга",
      email: "olga@example.com"
    });
    console.log(`Inserted with _id: ${result.insertedId}`);

    // Поиск
    const user = await users.findOne({ name: "Ольга" });
    console.log(user);

  } finally {
    await client.close();
  }
}

run().catch(console.dir);
```

### Python (PyMongo)

```python
from pymongo import MongoClient
from datetime import datetime

# Подключение
client = MongoClient("mongodb://localhost:27017/")
db = client["myapp"]
users = db["users"]

# Вставка
user = {
    "name": "Сергей",
    "email": "sergey@example.com",
    "created_at": datetime.now()
}
result = users.insert_one(user)
print(f"Inserted: {result.inserted_id}")

# Поиск
for user in users.find({"name": {"$regex": "^С"}}):
    print(user)

# Обновление
users.update_one(
    {"name": "Сергей"},
    {"$set": {"verified": True}}
)

# Закрытие соединения
client.close()
```

### Go (официальный драйвер)

```go
package main

import (
    "context"
    "fmt"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
    ctx := context.Background()

    client, err := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        panic(err)
    }
    defer client.Disconnect(ctx)

    collection := client.Database("myapp").Collection("users")

    // Вставка
    _, err = collection.InsertOne(ctx, bson.D{
        {Key: "name", Value: "Андрей"},
        {Key: "email", Value: "andrey@example.com"},
    })

    // Поиск
    var result bson.M
    err = collection.FindOne(ctx, bson.D{{Key: "name", Value: "Андрей"}}).Decode(&result)
    fmt.Println(result)
}
```

---

## Конфигурация MongoDB

### Файл конфигурации (mongod.conf)

```yaml
# Хранение данных
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2

# Сетевые настройки
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.100

# Логирование
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

# Безопасность
security:
  authorization: enabled

# Репликация
replication:
  replSetName: rs0

# Профилирование
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

---

## Полезные команды администрирования

```javascript
// Статистика базы данных
db.stats()

// Статистика коллекции
db.users.stats()

// Текущие операции
db.currentOp()

// Убить операцию
db.killOp(opId)

// Статус сервера
db.serverStatus()

// Статус репликации
rs.status()

// Создание индекса
db.users.createIndex({ email: 1 }, { unique: true })

// Список индексов
db.users.getIndexes()

// Удаление индекса
db.users.dropIndex("email_1")

// Компактификация коллекции
db.runCommand({ compact: "users" })

// Бэкап (из командной строки)
// mongodump --db myapp --out /backup/

// Восстановление
// mongorestore --db myapp /backup/myapp/
```

---

## Когда использовать MongoDB

### Подходит для:
- Каталоги продуктов с разными атрибутами
- Системы управления контентом (CMS)
- Real-time аналитика и логирование
- IoT и time-series данные
- Мобильные приложения
- Кеширование данных

### Не подходит для:
- Сложные транзакции между многими сущностями
- Системы с множеством связей (социальные графы)
- Системы с требованием strict schema
- Приложения, требующие сложные JOIN-запросы
