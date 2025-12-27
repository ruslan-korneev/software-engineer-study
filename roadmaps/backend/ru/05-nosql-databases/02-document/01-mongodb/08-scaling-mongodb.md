# Масштабирование MongoDB

## Введение

Масштабирование — это процесс увеличения производительности и ёмкости базы данных для обработки растущих нагрузок. MongoDB предоставляет мощные инструменты для масштабирования, включая репликацию (Replica Set) и шардирование (Sharding).

---

## 1. Вертикальное vs Горизонтальное масштабирование

### Вертикальное масштабирование (Scale Up)

Увеличение мощности одного сервера:
- Добавление CPU
- Увеличение RAM
- Более быстрые диски (SSD/NVMe)

```
┌─────────────────────────────────┐
│         Один сервер             │
│  ┌───────────────────────────┐  │
│  │   CPU: 4 → 16 → 64 ядер   │  │
│  │   RAM: 16GB → 128GB → 1TB │  │
│  │   Disk: HDD → SSD → NVMe  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**Преимущества:**
- Простота реализации
- Нет изменений в архитектуре приложения
- Нет проблем с распределёнными транзакциями

**Недостатки:**
- Физический предел оборудования
- Высокая стоимость мощных серверов
- Единая точка отказа (Single Point of Failure)

### Горизонтальное масштабирование (Scale Out)

Распределение нагрузки между несколькими серверами:

```
┌─────────────────────────────────────────────────────┐
│                   Кластер                           │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│  │ Сервер 1│   │ Сервер 2│   │ Сервер N│   ...     │
│  │  Shard  │   │  Shard  │   │  Shard  │           │
│  └─────────┘   └─────────┘   └─────────┘           │
└─────────────────────────────────────────────────────┘
```

**Преимущества:**
- Практически неограниченное масштабирование
- Отказоустойчивость
- Экономически эффективно (commodity hardware)

**Недостатки:**
- Сложность архитектуры
- Требует правильного проектирования (shard key)
- Сетевые задержки между узлами

---

## 2. Replica Set (Набор реплик)

### Архитектура Replica Set

Replica Set — группа mongod-процессов, которые поддерживают один и тот же набор данных.

```
┌─────────────────────────────────────────────────────────────┐
│                      REPLICA SET                            │
│                                                             │
│  ┌─────────────────┐                                        │
│  │     PRIMARY     │  ← Все операции записи                 │
│  │   (mongod:27017)│                                        │
│  └────────┬────────┘                                        │
│           │ Репликация (oplog)                              │
│     ┌─────┴─────┬─────────────┐                             │
│     ▼           ▼             ▼                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │SECONDARY │ │SECONDARY │ │ ARBITER  │                     │
│  │  :27018  │ │  :27019  │ │  :27020  │                     │
│  │(реплика) │ │(реплика) │ │(голосует)│                     │
│  └──────────┘ └──────────┘ └──────────┘                     │
│       ↑ Чтение (опционально)                                │
└─────────────────────────────────────────────────────────────┘
```

### Типы узлов

| Тип узла | Описание | Данные | Голосует |
|----------|----------|--------|----------|
| Primary | Принимает все записи | Да | Да |
| Secondary | Реплицирует данные, может обслуживать чтение | Да | Да |
| Arbiter | Только для голосования | Нет | Да |
| Hidden | Не видим для клиентов | Да | Да |
| Delayed | Задержанная реплика для recovery | Да | Нет |

### Инициализация Replica Set

```javascript
// Подключаемся к первому узлу
mongosh --port 27017

// Инициализируем Replica Set
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongodb-primary:27017", priority: 2 },
    { _id: 1, host: "mongodb-secondary1:27018", priority: 1 },
    { _id: 2, host: "mongodb-secondary2:27019", priority: 1 }
  ]
})

// Проверяем статус
rs.status()

// Добавляем arbiter (если нужен)
rs.addArb("mongodb-arbiter:27020")
```

### Конфигурация Replica Set

```javascript
// Изменение конфигурации
cfg = rs.conf()

// Установка приоритета (чем выше, тем больше шанс стать Primary)
cfg.members[0].priority = 10
cfg.members[1].priority = 5
cfg.members[2].priority = 1

// Hidden replica (для бэкапов/аналитики)
cfg.members[2].hidden = true
cfg.members[2].priority = 0

// Delayed replica (задержка 1 час)
cfg.members[2].secondaryDelaySecs = 3600
cfg.members[2].priority = 0

// Применяем конфигурацию
rs.reconfig(cfg)
```

### Выборы (Elections)

Выборы происходят автоматически, когда:
- Primary недоступен
- Primary выполняет `rs.stepDown()`
- Изменяется конфигурация с приоритетами

```
┌─────────────────────────────────────────────────────────┐
│                    ПРОЦЕСС ВЫБОРОВ                       │
│                                                          │
│  1. Primary недоступен                                   │
│     ┌─────────┐                                          │
│     │ PRIMARY │ ✗ (недоступен)                           │
│     └─────────┘                                          │
│                                                          │
│  2. Secondary замечает проблему (heartbeat timeout)      │
│     ┌───────────┐  ┌───────────┐                         │
│     │ SECONDARY │  │ SECONDARY │                         │
│     │  Кандидат │  │  Избирает │                         │
│     └─────┬─────┘  └─────┬─────┘                         │
│           │              │                               │
│           └──────┬───────┘                               │
│                  ▼                                       │
│  3. Голосование (требуется большинство)                  │
│     Votes: 2/3 → Кандидат становится Primary             │
│                                                          │
│  4. Новый Primary                                        │
│     ┌─────────┐                                          │
│     │   NEW   │                                          │
│     │ PRIMARY │                                          │
│     └─────────┘                                          │
└─────────────────────────────────────────────────────────┘
```

**Требования для выборов:**
- Большинство голосов (majority)
- Для 3 узлов: минимум 2 голоса
- Для 5 узлов: минимум 3 голоса
- Максимум 7 голосующих узлов

### Failover

```javascript
// Принудительный failover (на Primary)
rs.stepDown(60) // Primary станет Secondary на 60 секунд

// Заморозка Secondary (не может стать Primary)
rs.freeze(120) // Замораживаем на 120 секунд
```

### Read Preference (Предпочтение чтения)

```javascript
// Настройка в connection string
"mongodb://host1,host2,host3/?replicaSet=myRS&readPreference=secondaryPreferred"

// Или в коде (Node.js)
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred'
});
```

| Режим | Описание |
|-------|----------|
| primary | Только с Primary (по умолчанию) |
| primaryPreferred | Primary, если недоступен — Secondary |
| secondary | Только с Secondary |
| secondaryPreferred | Secondary, если недоступен — Primary |
| nearest | Ближайший по latency |

### Write Concern

```javascript
// Подтверждение записи от большинства
db.collection.insertOne(
  { item: "example" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)

// Подтверждение от всех узлов
db.collection.insertOne(
  { item: "example" },
  { writeConcern: { w: 3 } }
)
```

---

## 3. Sharding (Шардирование)

### Концепция

Sharding — это разделение данных между несколькими серверами для горизонтального масштабирования.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        SHARDED CLUSTER                                │
│                                                                       │
│                          ┌─────────┐                                  │
│                          │ mongos  │  ← Query Router                  │
│                          │ (роутер)│                                  │
│                          └────┬────┘                                  │
│                               │                                       │
│          ┌────────────────────┼────────────────────┐                  │
│          │                    │                    │                  │
│          ▼                    ▼                    ▼                  │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐           │
│  │  Config Server│   │  Config Server│   │  Config Server│           │
│  │   Replica Set │   │   Replica Set │   │   Replica Set │           │
│  │   (метаданные)│   │   (метаданные)│   │   (метаданные)│           │
│  └───────────────┘   └───────────────┘   └───────────────┘           │
│          │                    │                    │                  │
│          └────────────────────┼────────────────────┘                  │
│                               │                                       │
│          ┌────────────────────┼────────────────────┐                  │
│          │                    │                    │                  │
│          ▼                    ▼                    ▼                  │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐           │
│  │    Shard 1    │   │    Shard 2    │   │    Shard 3    │           │
│  │  Replica Set  │   │  Replica Set  │   │  Replica Set  │           │
│  │  (chunk A-M)  │   │  (chunk N-S)  │   │  (chunk T-Z)  │           │
│  └───────────────┘   └───────────────┘   └───────────────┘           │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Компоненты Sharded Cluster

#### 1. mongos (Query Router)

```
┌───────────────────────────────────────────┐
│                  mongos                   │
├───────────────────────────────────────────┤
│ • Точка входа для клиентов                │
│ • Маршрутизация запросов к шардам         │
│ • Объединение результатов                 │
│ • Кэширование метаданных                  │
│ • Stateless (можно запускать множество)   │
└───────────────────────────────────────────┘
```

```bash
# Запуск mongos
mongos --configdb configReplSet/cfg1:27019,cfg2:27019,cfg3:27019 \
       --port 27017 \
       --bind_ip localhost,192.168.1.100
```

#### 2. Config Servers (Серверы конфигурации)

```
┌───────────────────────────────────────────┐
│           Config Server Replica Set       │
├───────────────────────────────────────────┤
│ Хранят:                                   │
│ • Метаданные кластера                     │
│ • Маппинг chunk → shard                   │
│ • Настройки шардирования                  │
│ • Историю миграций                        │
│                                           │
│ Требования:                               │
│ • Обязательно Replica Set                 │
│ • Минимум 3 узла                          │
│ • Высокая доступность критична            │
└───────────────────────────────────────────┘
```

```bash
# Запуск config server
mongod --configsvr \
       --replSet configReplSet \
       --port 27019 \
       --dbpath /data/configdb \
       --bind_ip localhost,192.168.1.101
```

#### 3. Shards (Шарды)

```
┌───────────────────────────────────────────┐
│              Shard (Replica Set)          │
├───────────────────────────────────────────┤
│ • Хранит часть данных (chunks)            │
│ • Каждый шард — Replica Set               │
│ • Обрабатывает запросы к своим данным     │
│ • Участвует в агрегациях                  │
└───────────────────────────────────────────┘
```

```bash
# Запуск shard server
mongod --shardsvr \
       --replSet shard1ReplSet \
       --port 27018 \
       --dbpath /data/shard1 \
       --bind_ip localhost,192.168.1.102
```

### Настройка Sharded Cluster

```javascript
// 1. Подключаемся к mongos
mongosh --port 27017

// 2. Добавляем шарды
sh.addShard("shard1ReplSet/shard1-host1:27018,shard1-host2:27018,shard1-host3:27018")
sh.addShard("shard2ReplSet/shard2-host1:27018,shard2-host2:27018,shard2-host3:27018")
sh.addShard("shard3ReplSet/shard3-host1:27018,shard3-host2:27018,shard3-host3:27018")

// 3. Включаем шардирование для базы данных
sh.enableSharding("myDatabase")

// 4. Шардируем коллекцию
sh.shardCollection("myDatabase.myCollection", { userId: "hashed" })

// 5. Проверяем статус
sh.status()
```

---

## 4. Shard Key (Ключ шардирования)

### Что такое Shard Key

Shard Key — это поле (или комбинация полей), по которому MongoDB распределяет данные между шардами.

```
┌─────────────────────────────────────────────────────────────┐
│                    РАСПРЕДЕЛЕНИЕ ПО SHARD KEY                │
│                                                              │
│  Shard Key: userId                                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Shard 1    │  │   Shard 2    │  │   Shard 3    │       │
│  │              │  │              │  │              │       │
│  │ userId: 1-33 │  │ userId: 34-66│  │ userId: 67-99│       │
│  │              │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Типы Shard Key

#### Ranged Sharding (Диапазонное)

```javascript
// Создание ranged shard key
sh.shardCollection("myDB.orders", { orderDate: 1 })
```

```
┌─────────────────────────────────────────────────────────────┐
│                    RANGED SHARDING                           │
│                                                              │
│  Shard Key: orderDate                                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Shard 1    │  │   Shard 2    │  │   Shard 3    │       │
│  │              │  │              │  │              │       │
│  │  Jan - Apr   │  │  May - Aug   │  │  Sep - Dec   │       │
│  │              │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ✓ Хорошо для range queries                                  │
│  ✗ Риск "горячих" шардов при монотонных ключах              │
└─────────────────────────────────────────────────────────────┘
```

#### Hashed Sharding (Хэшированное)

```javascript
// Создание hashed shard key
sh.shardCollection("myDB.users", { _id: "hashed" })
```

```
┌─────────────────────────────────────────────────────────────┐
│                    HASHED SHARDING                           │
│                                                              │
│  Shard Key: _id (hashed)                                     │
│                                                              │
│  _id: 1 → hash → 0x7a3b → Shard 2                            │
│  _id: 2 → hash → 0x2f1c → Shard 1                            │
│  _id: 3 → hash → 0x9e4d → Shard 3                            │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Shard 1    │  │   Shard 2    │  │   Shard 3    │       │
│  │   ░░░░░░░░   │  │   ░░░░░░░░   │  │   ░░░░░░░░   │       │
│  │  Равномерно  │  │  распределено│  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ✓ Равномерное распределение                                 │
│  ✗ Неэффективные range queries                               │
└─────────────────────────────────────────────────────────────┘
```

#### Compound Shard Key (Составной)

```javascript
// Составной ключ
sh.shardCollection("myDB.logs", { tenantId: 1, timestamp: 1 })
```

### Выбор Shard Key

#### Критерии хорошего Shard Key

| Критерий | Описание |
|----------|----------|
| Кардинальность | Много уникальных значений |
| Частота | Значения распределены равномерно |
| Неизменяемость | Значение не меняется после создания |
| Query Isolation | Запросы попадают на минимум шардов |

```javascript
// ❌ ПЛОХОЙ Shard Key
// Низкая кардинальность
sh.shardCollection("myDB.users", { country: 1 })

// ❌ ПЛОХОЙ Shard Key
// Монотонно возрастающий
sh.shardCollection("myDB.orders", { createdAt: 1 })

// ✓ ХОРОШИЙ Shard Key
// Высокая кардинальность + изоляция запросов
sh.shardCollection("myDB.orders", { customerId: "hashed" })

// ✓ ХОРОШИЙ Shard Key
// Составной для баланса
sh.shardCollection("myDB.events", { tenantId: 1, eventId: "hashed" })
```

### Chunks (Части данных)

```
┌─────────────────────────────────────────────────────────────┐
│                         CHUNKS                               │
│                                                              │
│  Chunk = диапазон значений shard key                         │
│  Размер по умолчанию: 128 MB                                 │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ Chunk 1 │  │ Chunk 2 │  │ Chunk 3 │  │ Chunk 4 │         │
│  │ [A-D]   │  │ [E-H]   │  │ [I-L]   │  │ [M-P]   │         │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │
│       ↓            ↓            ↓            ↓               │
│    Shard 1      Shard 2      Shard 1      Shard 3           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```javascript
// Просмотр информации о chunks
use config
db.chunks.find({ ns: "myDB.myCollection" }).pretty()

// Изменение размера chunk (в MB)
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 64 } },
  { upsert: true }
)
```

---

## 5. Балансировка данных

### Balancer

Balancer — фоновый процесс, который автоматически перемещает chunks между шардами для равномерного распределения.

```
┌─────────────────────────────────────────────────────────────┐
│                    БАЛАНСИРОВКА                              │
│                                                              │
│  ДО БАЛАНСИРОВКИ:                                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Shard 1   │  │  Shard 2   │  │  Shard 3   │             │
│  │ ████████   │  │ ██         │  │ ███        │             │
│  │ 80 chunks  │  │ 20 chunks  │  │ 30 chunks  │             │
│  └────────────┘  └────────────┘  └────────────┘             │
│                                                              │
│                      ▼ Balancer                              │
│                                                              │
│  ПОСЛЕ БАЛАНСИРОВКИ:                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Shard 1   │  │  Shard 2   │  │  Shard 3   │             │
│  │ ████       │  │ ████       │  │ ████       │             │
│  │ 43 chunks  │  │ 43 chunks  │  │ 44 chunks  │             │
│  └────────────┘  └────────────┘  └────────────┘             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Управление Balancer

```javascript
// Проверка статуса balancer
sh.isBalancerRunning()

// Остановка balancer
sh.stopBalancer()

// Запуск balancer
sh.startBalancer()

// Расписание балансировки (только ночью)
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: {
    activeWindow: {
      start: "02:00",
      stop: "06:00"
    }
  }},
  { upsert: true }
)

// Отключение балансировки для конкретной коллекции
sh.disableBalancing("myDB.myCollection")
sh.enableBalancing("myDB.myCollection")
```

### Миграция chunks

```javascript
// Ручное перемещение chunk
sh.moveChunk(
  "myDB.myCollection",
  { shardKey: "valueInChunk" },
  "shard2"
)

// Просмотр истории миграций
use config
db.changelog.find({ what: "moveChunk.commit" }).sort({ time: -1 }).limit(10)
```

---

## 6. Зоны (Zones) для географического распределения

### Концепция Zones

Zones позволяют контролировать, на каких шардах хранятся определённые данные.

```
┌─────────────────────────────────────────────────────────────┐
│                    ZONES (ЗОНЫ)                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 ПРИЛОЖЕНИЕ                           │    │
│  └───────────────────────┬─────────────────────────────┘    │
│                          │                                   │
│              ┌───────────┴───────────┐                       │
│              ▼                       ▼                       │
│  ┌──────────────────┐    ┌──────────────────┐               │
│  │  Zone: "europe"  │    │  Zone: "america" │               │
│  ├──────────────────┤    ├──────────────────┤               │
│  │ region: "EU"     │    │ region: "US"     │               │
│  ├──────────────────┤    ├──────────────────┤               │
│  │   Shard EU-1     │    │   Shard US-1     │               │
│  │   Shard EU-2     │    │   Shard US-2     │               │
│  │  (Франкфурт)     │    │  (Вирджиния)     │               │
│  └──────────────────┘    └──────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Настройка Zones

```javascript
// 1. Добавляем шарды в зоны
sh.addShardTag("shard-eu-1", "europe")
sh.addShardTag("shard-eu-2", "europe")
sh.addShardTag("shard-us-1", "america")
sh.addShardTag("shard-us-2", "america")

// 2. Определяем диапазоны для зон
sh.updateZoneKeyRange(
  "myDB.users",
  { region: "EU", _id: MinKey },
  { region: "EU", _id: MaxKey },
  "europe"
)

sh.updateZoneKeyRange(
  "myDB.users",
  { region: "US", _id: MinKey },
  { region: "US", _id: MaxKey },
  "america"
)

// Shard key должен включать поле region
sh.shardCollection("myDB.users", { region: 1, _id: 1 })

// 3. Проверка зон
sh.status()
```

### Примеры использования Zones

#### Географическое распределение (Data Residency)

```javascript
// Данные пользователей EU остаются в EU
sh.updateZoneKeyRange(
  "app.userData",
  { country: "DE", userId: MinKey },
  { country: "DE", userId: MaxKey },
  "europe-zone"
)
```

#### Tiered Storage (Горячие/холодные данные)

```javascript
// Недавние данные на быстрых SSD
sh.addShardTag("shard-ssd-1", "hot")

// Старые данные на HDD
sh.addShardTag("shard-hdd-1", "cold")

sh.updateZoneKeyRange(
  "logs.events",
  { timestamp: ISODate("2024-01-01") },
  { timestamp: MaxKey },
  "hot"
)

sh.updateZoneKeyRange(
  "logs.events",
  { timestamp: MinKey },
  { timestamp: ISODate("2024-01-01") },
  "cold"
)
```

---

## 7. Мониторинг кластера

### Команды мониторинга

```javascript
// Статус Replica Set
rs.status()
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()

// Статус Sharded Cluster
sh.status()
sh.getBalancerState()

// Информация о сервере
db.serverStatus()
db.stats()
db.collection.stats()

// Текущие операции
db.currentOp()
db.currentOp({ "active": true, "secs_running": { "$gt": 3 } })

// Профилирование
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(5)
```

### Ключевые метрики

```javascript
// Метрики репликации
db.adminCommand({ replSetGetStatus: 1 }).members.forEach(m => {
  print(`${m.name}: state=${m.stateStr}, lag=${m.optimeDate}`)
})

// Метрики шардирования
use config
db.chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### Мониторинг с MongoDB Atlas / Cloud Manager

```
┌─────────────────────────────────────────────────────────────┐
│              DASHBOARD МОНИТОРИНГА                           │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   CPU       │  │   Memory    │  │   Disk I/O  │          │
│  │   ████▓░░   │  │   █████░░   │  │   ██▓░░░░   │          │
│  │   67%       │  │   78%       │  │   32%       │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Replication Lag                                     │    │
│  │  Secondary1: 0.5s ████▓░░░░░                         │    │
│  │  Secondary2: 1.2s ██████▓░░░                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Chunk Distribution                                  │    │
│  │  Shard1: 245 ████████████                            │    │
│  │  Shard2: 238 ███████████                             │    │
│  │  Shard3: 241 ████████████                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Алерты и уведомления

```javascript
// Скрипт проверки здоровья кластера
function checkClusterHealth() {
  const rsStatus = rs.status();

  // Проверка Primary
  const primary = rsStatus.members.find(m => m.stateStr === "PRIMARY");
  if (!primary) {
    print("ALERT: No PRIMARY found!");
    return false;
  }

  // Проверка отставания реплик
  rsStatus.members.forEach(m => {
    if (m.stateStr === "SECONDARY") {
      const lag = (primary.optimeDate - m.optimeDate) / 1000;
      if (lag > 10) {
        print(`WARNING: ${m.name} lag is ${lag} seconds`);
      }
    }
  });

  return true;
}
```

---

## 8. Best Practices при масштабировании

### Replica Set Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│               REPLICA SET BEST PRACTICES                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✓ Используйте нечётное число узлов (3, 5, 7)               │
│    - Обеспечивает majority при выборах                       │
│                                                              │
│  ✓ Размещайте узлы в разных дата-центрах                    │
│    - Минимум 2 дата-центра                                   │
│    - Majority узлов в основном дата-центре                   │
│                                                              │
│  ✓ Используйте Priority для контроля Primary                 │
│    - Высокий приоритет для мощных серверов                   │
│                                                              │
│  ✓ Настройте Hidden реплику для бэкапов                      │
│    - Не влияет на производительность                         │
│                                                              │
│  ✓ Delayed replica для защиты от ошибок                      │
│    - Задержка 1-24 часа                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Sharding Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│                 SHARDING BEST PRACTICES                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✓ Планируйте Shard Key заранее                              │
│    - Изменить shard key очень сложно                         │
│    - Тестируйте на реальных данных                           │
│                                                              │
│  ✓ Используйте составной shard key                           │
│    - { tenantId: 1, timestamp: 1 }                           │
│    - Баланс между распределением и локальностью              │
│                                                              │
│  ✓ Минимум 3 шарда                                           │
│    - 2 шарда — минимальная отказоустойчивость                │
│    - Планируйте рост заранее                                 │
│                                                              │
│  ✓ Используйте mongos на application servers                 │
│    - Уменьшает сетевые хопы                                  │
│    - Локальный кэш метаданных                                │
│                                                              │
│  ✓ Мониторьте chunk distribution                             │
│    - Неравномерность = проблемы                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Конфигурация для продакшена

```yaml
# mongod.conf для shard server
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # 50% RAM

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27018
  bindIp: 0.0.0.0

replication:
  replSetName: shard1RS

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

### Sizing и планирование

```
┌─────────────────────────────────────────────────────────────┐
│                    SIZING GUIDELINES                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  RAM:                                                        │
│  • Working Set должен помещаться в RAM                       │
│  • WiredTiger cache = 50% RAM (по умолчанию)                 │
│  • Минимум: 8GB для продакшена                               │
│                                                              │
│  CPU:                                                        │
│  • 1 ядро на 1000 операций/сек (примерно)                    │
│  • Больше ядер для агрегаций                                 │
│                                                              │
│  Storage:                                                    │
│  • SSD для WiredTiger                                        │
│  • RAID 10 для высокой доступности                           │
│  • Запас 40% для роста + временных файлов                    │
│                                                              │
│  Network:                                                    │
│  • 10 Gbps между узлами кластера                             │
│  • Низкая latency (<1ms) внутри дата-центра                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Типичные ошибки

### Ошибка 1: Неправильный выбор Shard Key

```javascript
// ❌ ОШИБКА: Монотонно возрастающий ключ
sh.shardCollection("logs.events", { timestamp: 1 })
// Результат: Все записи идут на один шард

// ✓ РЕШЕНИЕ: Hashed или составной ключ
sh.shardCollection("logs.events", { timestamp: "hashed" })
// или
sh.shardCollection("logs.events", { appId: 1, timestamp: 1 })
```

### Ошибка 2: Jumbo Chunks

```
┌─────────────────────────────────────────────────────────────┐
│                    JUMBO CHUNKS                              │
│                                                              │
│  Проблема: Chunk больше максимального размера               │
│  Причина: Все документы имеют одинаковый shard key          │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │  Chunk [userId: 123]                              │       │
│  │  ██████████████████████████████████████████████   │       │
│  │  500 MB (не может быть разделён!)                 │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Решение: Используйте shard key с высокой кардинальностью   │
└─────────────────────────────────────────────────────────────┘
```

```javascript
// Проверка jumbo chunks
use config
db.chunks.find({ jumbo: true })

// Очистка флага jumbo (после устранения причины)
db.chunks.updateOne(
  { _id: "myDB.myCollection-shardKey_value" },
  { $unset: { jumbo: "" } }
)
```

### Ошибка 3: Scatter-Gather запросы

```javascript
// ❌ ОШИБКА: Запрос без shard key (идёт на все шарды)
db.orders.find({ status: "pending" })

// ✓ РЕШЕНИЕ: Включайте shard key в запрос
db.orders.find({ customerId: "123", status: "pending" })
```

### Ошибка 4: Чётное количество узлов

```
┌─────────────────────────────────────────────────────────────┐
│                    SPLIT BRAIN                               │
│                                                              │
│  2 узла: При потере связи оба считают себя Primary          │
│                                                              │
│  ┌─────────┐         ✗         ┌─────────┐                  │
│  │ Node 1  │ ═══════════════ │ Node 2  │                  │
│  │ Primary?│    Network       │ Primary?│                  │
│  └─────────┘    Partition     └─────────┘                  │
│                                                              │
│  Решение: Используйте 3, 5 или 7 узлов                       │
│           Или добавьте Arbiter                               │
└─────────────────────────────────────────────────────────────┘
```

### Ошибка 5: Игнорирование replication lag

```javascript
// ❌ ОШИБКА: Чтение сразу после записи с secondary
await db.collection.insertOne({ data: "test" })
// На другом соединении с readPreference: secondary
await db.collection.findOne({ data: "test" }) // Может не найти!

// ✓ РЕШЕНИЕ: Используйте write concern + read concern
await db.collection.insertOne(
  { data: "test" },
  { writeConcern: { w: "majority" } }
)
// Читаем с гарантией
await db.collection.findOne(
  { data: "test" },
  { readConcern: { level: "majority" } }
)
```

### Ошибка 6: Балансировка во время пиковых нагрузок

```javascript
// ✓ Настройте окно балансировки
use config
db.settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: { start: "02:00", stop: "06:00" }
    }
  },
  { upsert: true }
)
```

### Ошибка 7: Отсутствие индексов на shard key

```javascript
// Shard key ДОЛЖЕН иметь индекс
db.collection.createIndex({ shardKeyField: 1 })
// или для hashed
db.collection.createIndex({ shardKeyField: "hashed" })

// Проверка индексов
db.collection.getIndexes()
```

---

## Итоги

| Концепция | Назначение | Ключевые моменты |
|-----------|------------|------------------|
| Replica Set | Высокая доступность | Primary/Secondary, автоматический failover |
| Sharding | Горизонтальное масштабирование | mongos, config servers, shards |
| Shard Key | Распределение данных | Высокая кардинальность, неизменяемость |
| Balancer | Автоматическая балансировка | Окна обслуживания, мониторинг |
| Zones | Географическое распределение | Data residency, tiered storage |

**Рекомендуемый порядок масштабирования:**
1. Начните с Replica Set для отказоустойчивости
2. Оптимизируйте индексы и запросы
3. Вертикальное масштабирование (если возможно)
4. Sharding при необходимости горизонтального масштабирования
