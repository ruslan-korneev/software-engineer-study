# Транзакции в MongoDB

## Введение

Транзакции — это механизм, обеспечивающий атомарность группы операций. До версии 4.0 MongoDB гарантировала атомарность только на уровне одного документа. Начиная с версии 4.0 появилась поддержка многодокументных транзакций для replica sets, а с версии 4.2 — для sharded clusters.

---

## ACID в MongoDB

MongoDB поддерживает ACID-гарантии:

| Свойство | Описание | Реализация в MongoDB |
|----------|----------|----------------------|
| **Atomicity** (Атомарность) | Все операции выполняются или откатываются целиком | Транзакции и однодокументные операции |
| **Consistency** (Согласованность) | Данные переходят из одного валидного состояния в другое | Schema validation, unique indexes |
| **Isolation** (Изолированность) | Параллельные транзакции не влияют друг на друга | Snapshot isolation |
| **Durability** (Долговечность) | Подтверждённые изменения сохраняются | Write concern, journaling |

### Уровни изоляции

MongoDB использует **snapshot isolation** для транзакций:
- Транзакция видит консистентный снимок данных на момент начала
- Изменения других транзакций не видны до их коммита
- Предотвращает dirty reads, non-repeatable reads и phantom reads

---

## Однодокументные атомарные операции

MongoDB гарантирует атомарность операций над **одним документом**, включая вложенные поддокументы и массивы.

```javascript
// Атомарное обновление нескольких полей в одном документе
db.accounts.updateOne(
  { _id: "account123" },
  {
    $inc: { balance: -100 },
    $push: {
      transactions: {
        type: "withdrawal",
        amount: 100,
        date: new Date()
      }
    },
    $set: { lastModified: new Date() }
  }
);

// Атомарный findAndModify
db.inventory.findOneAndUpdate(
  { item: "laptop", quantity: { $gte: 1 } },
  {
    $inc: { quantity: -1 },
    $push: { reservations: { orderId: "ORD123", date: new Date() } }
  },
  { returnDocument: "after" }
);
```

### Когда достаточно однодокументных операций

- Обновление связанных данных внутри одного документа
- Денормализованные данные (встроенные документы)
- Операции с массивами внутри документа
- Счётчики и инкременты

---

## Многодокументные транзакции (MongoDB 4.0+)

Многодокументные транзакции позволяют выполнять атомарные операции над несколькими документами, коллекциями и даже базами данных.

### Требования

- **Replica Set** (минимум 1 primary + 1 secondary) или **Sharded Cluster**
- MongoDB 4.0+ для replica sets
- MongoDB 4.2+ для sharded clusters
- WiredTiger storage engine

### Базовый синтаксис

```javascript
// Создание сессии и транзакции
const session = db.getMongo().startSession();

session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

try {
  const accounts = session.getDatabase("bank").accounts;

  // Операции внутри транзакции
  accounts.updateOne(
    { _id: "sender" },
    { $inc: { balance: -100 } },
    { session }
  );

  accounts.updateOne(
    { _id: "receiver" },
    { $inc: { balance: 100 } },
    { session }
  );

  // Подтверждение транзакции
  session.commitTransaction();
  print("Transaction committed");

} catch (error) {
  // Откат транзакции
  session.abortTransaction();
  print("Transaction aborted: " + error.message);

} finally {
  session.endSession();
}
```

---

## Синтаксис и API транзакций

### Основные методы

```javascript
// 1. startSession() - создание сессии
const session = client.startSession({
  defaultTransactionOptions: {
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
    readPreference: "primary",
    maxCommitTimeMS: 1000
  }
});

// 2. startTransaction() - начало транзакции
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

// 3. commitTransaction() - подтверждение
await session.commitTransaction();

// 4. abortTransaction() - откат
await session.abortTransaction();

// 5. endSession() - завершение сессии
session.endSession();
```

### Callback API (рекомендуемый способ)

MongoDB предоставляет удобный callback API с автоматическим retry:

```javascript
// Node.js Driver
const { MongoClient } = require('mongodb');

async function transferMoney(client, fromAccount, toAccount, amount) {
  const session = client.startSession();

  const transactionOptions = {
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
    readPreference: 'primary'
  };

  try {
    // withTransaction автоматически обрабатывает retry
    const result = await session.withTransaction(async () => {
      const accounts = client.db('bank').collection('accounts');

      // Проверяем баланс отправителя
      const sender = await accounts.findOne(
        { _id: fromAccount },
        { session }
      );

      if (!sender || sender.balance < amount) {
        throw new Error('Insufficient funds');
      }

      // Списываем с отправителя
      await accounts.updateOne(
        { _id: fromAccount },
        {
          $inc: { balance: -amount },
          $push: {
            history: {
              type: 'transfer_out',
              amount,
              to: toAccount,
              date: new Date()
            }
          }
        },
        { session }
      );

      // Начисляем получателю
      await accounts.updateOne(
        { _id: toAccount },
        {
          $inc: { balance: amount },
          $push: {
            history: {
              type: 'transfer_in',
              amount,
              from: fromAccount,
              date: new Date()
            }
          }
        },
        { session }
      );

      return { success: true, amount };

    }, transactionOptions);

    console.log('Transfer completed:', result);
    return result;

  } catch (error) {
    console.error('Transfer failed:', error.message);
    throw error;

  } finally {
    await session.endSession();
  }
}
```

### Python Driver (PyMongo)

```python
from pymongo import MongoClient
from pymongo.errors import PyMongoError

def transfer_money(client, from_account, to_account, amount):
    with client.start_session() as session:
        def callback(session):
            accounts = client.bank.accounts

            # Проверяем баланс
            sender = accounts.find_one(
                {"_id": from_account},
                session=session
            )

            if not sender or sender.get("balance", 0) < amount:
                raise ValueError("Insufficient funds")

            # Списание
            accounts.update_one(
                {"_id": from_account},
                {"$inc": {"balance": -amount}},
                session=session
            )

            # Начисление
            accounts.update_one(
                {"_id": to_account},
                {"$inc": {"balance": amount}},
                session=session
            )

            return True

        try:
            result = session.with_transaction(
                callback,
                read_concern={"level": "snapshot"},
                write_concern={"w": "majority"}
            )
            return result
        except PyMongoError as e:
            print(f"Transaction failed: {e}")
            raise
```

---

## Read/Write Concerns в транзакциях

### Read Concern

Определяет уровень согласованности читаемых данных:

| Level | Описание | Использование |
|-------|----------|---------------|
| `local` | Читает локальные данные ноды | По умолчанию для standalone |
| `available` | Самые доступные данные (sharded) | Низкая задержка |
| `majority` | Данные, подтверждённые большинством | Надёжное чтение |
| `snapshot` | Снимок данных на момент начала | **Рекомендуется для транзакций** |
| `linearizable` | Строго линейная согласованность | Критичные операции |

```javascript
session.startTransaction({
  readConcern: { level: "snapshot" }
});
```

### Write Concern

Определяет гарантии записи:

| Level | Описание |
|-------|----------|
| `{ w: 1 }` | Подтверждение от primary |
| `{ w: "majority" }` | Подтверждение от большинства нод |
| `{ w: <число> }` | Подтверждение от N нод |
| `{ j: true }` | Запись в journal |
| `{ wtimeout: ms }` | Таймаут ожидания |

```javascript
session.startTransaction({
  writeConcern: { w: "majority", j: true }
});
```

### Рекомендуемые настройки

```javascript
// Для критичных операций (финансы, заказы)
{
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority", j: true }
}

// Для менее критичных операций
{
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
}
```

---

## Ограничения транзакций

### Технические ограничения

1. **Время жизни транзакции**
   - Максимум 60 секунд по умолчанию (`transactionLifetimeLimitSeconds`)
   - Может быть увеличено через конфигурацию

2. **Размер oplog**
   - Все изменения должны поместиться в oplog entry (16MB)
   - Для больших транзакций нужен больший oplog

3. **Операции, запрещённые в транзакциях**
   ```javascript
   // Нельзя использовать внутри транзакций:
   db.createCollection()        // Создание коллекций
   db.collection.createIndex()  // Создание индексов
   db.collection.drop()         // Удаление коллекций
   db.dropDatabase()            // Удаление базы данных
   ```

4. **Sharded Clusters (дополнительно)**
   - Нельзя писать в `config` и `admin` базы
   - Cross-shard транзакции имеют больший overhead

### Производительность

- Транзакции добавляют overhead по сравнению с одиночными операциями
- Long-running транзакции блокируют ресурсы
- Конфликты (write conflicts) требуют retry

```javascript
// Мониторинг транзакций
db.adminCommand({
  currentOp: true,
  $or: [
    { "transaction": { $exists: true } }
  ]
});

// Статистика транзакций
db.serverStatus().transactions
```

---

## Retry логика

### Transient Transaction Errors

Временные ошибки, после которых транзакцию можно повторить целиком:

```javascript
async function runTransactionWithRetry(session, txnFunc) {
  while (true) {
    try {
      await txnFunc(session);
      return;
    } catch (error) {
      // Повторяем при временных ошибках
      if (error.hasErrorLabel('TransientTransactionError')) {
        console.log('TransientTransactionError, retrying...');
        continue;
      }
      throw error;
    }
  }
}
```

### Unknown Transaction Commit Errors

Ошибки при коммите, когда неизвестно, был ли коммит успешен:

```javascript
async function commitWithRetry(session) {
  while (true) {
    try {
      await session.commitTransaction();
      console.log('Transaction committed');
      return;
    } catch (error) {
      if (error.hasErrorLabel('UnknownTransactionCommitResult')) {
        console.log('UnknownTransactionCommitResult, retrying commit...');
        continue;
      }
      throw error;
    }
  }
}
```

### Полная retry логика

```javascript
async function runTransaction(client, txnFunc) {
  const session = client.startSession();

  try {
    await runTransactionWithRetry(session, async (session) => {
      session.startTransaction({
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' }
      });

      await txnFunc(session);
      await commitWithRetry(session);
    });
  } finally {
    await session.endSession();
  }
}

// Использование
await runTransaction(client, async (session) => {
  await db.collection('orders').insertOne(
    { item: 'laptop', qty: 1 },
    { session }
  );
});
```

> **Примечание**: Метод `withTransaction()` автоматически обрабатывает retry логику, поэтому рекомендуется использовать его.

---

## Практические примеры

### Пример 1: Перевод денег между счетами

```javascript
// Node.js с полной обработкой ошибок
const { MongoClient } = require('mongodb');

class BankService {
  constructor(client) {
    this.client = client;
    this.accounts = client.db('bank').collection('accounts');
    this.transactions = client.db('bank').collection('transactions');
  }

  async transfer(fromId, toId, amount, description = '') {
    const session = this.client.startSession();

    try {
      const result = await session.withTransaction(async () => {
        // 1. Проверяем существование счетов
        const [sender, receiver] = await Promise.all([
          this.accounts.findOne({ _id: fromId }, { session }),
          this.accounts.findOne({ _id: toId }, { session })
        ]);

        if (!sender) {
          throw new Error(`Sender account ${fromId} not found`);
        }
        if (!receiver) {
          throw new Error(`Receiver account ${toId} not found`);
        }

        // 2. Проверяем баланс
        if (sender.balance < amount) {
          throw new Error(
            `Insufficient funds: ${sender.balance} < ${amount}`
          );
        }

        // 3. Создаём запись о транзакции
        const txnId = new ObjectId();
        const txnRecord = {
          _id: txnId,
          from: fromId,
          to: toId,
          amount,
          description,
          status: 'completed',
          createdAt: new Date()
        };

        await this.transactions.insertOne(txnRecord, { session });

        // 4. Обновляем балансы
        await Promise.all([
          this.accounts.updateOne(
            { _id: fromId },
            {
              $inc: { balance: -amount },
              $push: {
                transactionHistory: {
                  txnId,
                  type: 'debit',
                  amount: -amount,
                  counterparty: toId,
                  date: new Date()
                }
              }
            },
            { session }
          ),
          this.accounts.updateOne(
            { _id: toId },
            {
              $inc: { balance: amount },
              $push: {
                transactionHistory: {
                  txnId,
                  type: 'credit',
                  amount: amount,
                  counterparty: fromId,
                  date: new Date()
                }
              }
            },
            { session }
          )
        ]);

        return { txnId, status: 'completed' };

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' }
      });

      return result;

    } catch (error) {
      console.error('Transfer failed:', error);
      throw error;

    } finally {
      await session.endSession();
    }
  }
}

// Использование
const client = new MongoClient(uri);
await client.connect();

const bank = new BankService(client);
const result = await bank.transfer(
  'ACC001',
  'ACC002',
  500,
  'Payment for services'
);
```

### Пример 2: Резервирование товара

```javascript
class InventoryService {
  constructor(client) {
    this.client = client;
    this.products = client.db('shop').collection('products');
    this.reservations = client.db('shop').collection('reservations');
  }

  async reserveItems(orderId, items, expiresInMinutes = 15) {
    const session = this.client.startSession();

    try {
      const result = await session.withTransaction(async () => {
        const reservedItems = [];
        const expiresAt = new Date(
          Date.now() + expiresInMinutes * 60 * 1000
        );

        for (const item of items) {
          // Проверяем и уменьшаем доступное количество
          const updateResult = await this.products.findOneAndUpdate(
            {
              _id: item.productId,
              availableQuantity: { $gte: item.quantity }
            },
            {
              $inc: {
                availableQuantity: -item.quantity,
                reservedQuantity: item.quantity
              }
            },
            {
              session,
              returnDocument: 'after'
            }
          );

          if (!updateResult) {
            throw new Error(
              `Insufficient stock for product ${item.productId}`
            );
          }

          reservedItems.push({
            productId: item.productId,
            productName: updateResult.name,
            quantity: item.quantity,
            price: updateResult.price
          });
        }

        // Создаём запись о резервировании
        const reservation = {
          _id: new ObjectId(),
          orderId,
          items: reservedItems,
          status: 'active',
          createdAt: new Date(),
          expiresAt
        };

        await this.reservations.insertOne(reservation, { session });

        return reservation;

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' }
      });

      return result;

    } finally {
      await session.endSession();
    }
  }

  async confirmReservation(reservationId) {
    const session = this.client.startSession();

    try {
      await session.withTransaction(async () => {
        const reservation = await this.reservations.findOne(
          { _id: reservationId, status: 'active' },
          { session }
        );

        if (!reservation) {
          throw new Error('Reservation not found or already processed');
        }

        // Уменьшаем зарезервированное количество
        for (const item of reservation.items) {
          await this.products.updateOne(
            { _id: item.productId },
            { $inc: { reservedQuantity: -item.quantity } },
            { session }
          );
        }

        // Обновляем статус резервирования
        await this.reservations.updateOne(
          { _id: reservationId },
          {
            $set: {
              status: 'confirmed',
              confirmedAt: new Date()
            }
          },
          { session }
        );

      });

    } finally {
      await session.endSession();
    }
  }

  async cancelReservation(reservationId) {
    const session = this.client.startSession();

    try {
      await session.withTransaction(async () => {
        const reservation = await this.reservations.findOne(
          { _id: reservationId, status: 'active' },
          { session }
        );

        if (!reservation) {
          throw new Error('Reservation not found or already processed');
        }

        // Возвращаем товары в доступное количество
        for (const item of reservation.items) {
          await this.products.updateOne(
            { _id: item.productId },
            {
              $inc: {
                availableQuantity: item.quantity,
                reservedQuantity: -item.quantity
              }
            },
            { session }
          );
        }

        // Обновляем статус
        await this.reservations.updateOne(
          { _id: reservationId },
          {
            $set: {
              status: 'cancelled',
              cancelledAt: new Date()
            }
          },
          { session }
        );

      });

    } finally {
      await session.endSession();
    }
  }
}
```

### Пример 3: Создание заказа (Order Saga)

```javascript
class OrderService {
  constructor(client) {
    this.client = client;
    this.orders = client.db('shop').collection('orders');
    this.inventory = new InventoryService(client);
    this.payments = client.db('shop').collection('payments');
  }

  async createOrder(userId, items, paymentMethod) {
    const session = this.client.startSession();

    try {
      const result = await session.withTransaction(async () => {
        // 1. Рассчитываем сумму
        const products = this.client.db('shop').collection('products');
        let totalAmount = 0;
        const orderItems = [];

        for (const item of items) {
          const product = await products.findOne(
            { _id: item.productId },
            { session }
          );

          if (!product) {
            throw new Error(`Product ${item.productId} not found`);
          }

          if (product.availableQuantity < item.quantity) {
            throw new Error(`Insufficient stock for ${product.name}`);
          }

          orderItems.push({
            productId: product._id,
            name: product.name,
            price: product.price,
            quantity: item.quantity,
            subtotal: product.price * item.quantity
          });

          totalAmount += product.price * item.quantity;
        }

        // 2. Резервируем товары
        for (const item of items) {
          await products.updateOne(
            {
              _id: item.productId,
              availableQuantity: { $gte: item.quantity }
            },
            {
              $inc: {
                availableQuantity: -item.quantity,
                reservedQuantity: item.quantity
              }
            },
            { session }
          );
        }

        // 3. Создаём платёж
        const payment = {
          _id: new ObjectId(),
          userId,
          amount: totalAmount,
          method: paymentMethod,
          status: 'pending',
          createdAt: new Date()
        };

        await this.payments.insertOne(payment, { session });

        // 4. Создаём заказ
        const order = {
          _id: new ObjectId(),
          userId,
          items: orderItems,
          totalAmount,
          paymentId: payment._id,
          status: 'created',
          createdAt: new Date()
        };

        await this.orders.insertOne(order, { session });

        return {
          orderId: order._id,
          paymentId: payment._id,
          totalAmount,
          items: orderItems
        };

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' }
      });

      return result;

    } catch (error) {
      console.error('Order creation failed:', error);
      throw error;

    } finally {
      await session.endSession();
    }
  }
}
```

---

## Best Practices

### 1. Используйте транзакции только когда необходимо

```javascript
// ❌ Плохо: транзакция для одного документа
await session.withTransaction(async () => {
  await users.updateOne({ _id: userId }, { $set: { name: 'John' } });
});

// ✅ Хорошо: прямое обновление
await users.updateOne({ _id: userId }, { $set: { name: 'John' } });

// ✅ Хорошо: транзакция для связанных данных
await session.withTransaction(async () => {
  await accounts.updateOne({ _id: from }, { $inc: { balance: -100 } });
  await accounts.updateOne({ _id: to }, { $inc: { balance: 100 } });
});
```

### 2. Минимизируйте время транзакции

```javascript
// ❌ Плохо: длительные операции внутри транзакции
await session.withTransaction(async () => {
  const data = await fetchExternalAPI(); // Долгая операция!
  await collection.insertOne(data, { session });
});

// ✅ Хорошо: подготовка данных до транзакции
const data = await fetchExternalAPI();
await session.withTransaction(async () => {
  await collection.insertOne(data, { session });
});
```

### 3. Используйте withTransaction() вместо ручного управления

```javascript
// ❌ Плохо: ручное управление без retry
session.startTransaction();
try {
  // операции
  await session.commitTransaction();
} catch (e) {
  await session.abortTransaction();
}

// ✅ Хорошо: автоматический retry
await session.withTransaction(async () => {
  // операции
});
```

### 4. Правильная обработка ошибок

```javascript
async function safeTransaction(client, operation) {
  const session = client.startSession();

  try {
    const result = await session.withTransaction(
      operation,
      {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' },
        maxCommitTimeMS: 5000
      }
    );
    return { success: true, result };

  } catch (error) {
    // Логируем детали ошибки
    console.error('Transaction failed:', {
      message: error.message,
      code: error.code,
      labels: error.errorLabels
    });

    return {
      success: false,
      error: error.message,
      retryable: error.hasErrorLabel?.('TransientTransactionError')
    };

  } finally {
    await session.endSession();
  }
}
```

### 5. Мониторинг транзакций

```javascript
// Настройка профилирования
db.setProfilingLevel(1, { slowms: 100 });

// Просмотр медленных транзакций
db.system.profile.find({
  'command.startTransaction': { $exists: true }
}).sort({ millis: -1 }).limit(10);

// Метрики транзакций
const stats = db.serverStatus().transactions;
console.log({
  totalStarted: stats.totalStarted,
  totalCommitted: stats.totalCommitted,
  totalAborted: stats.totalAborted,
  currentActive: stats.currentActive
});
```

### 6. Проектируйте данные для минимизации транзакций

```javascript
// ❌ Требует транзакции: данные в разных документах
// orders: { _id, items: [itemId1, itemId2] }
// items: { _id, orderId, product, price }

// ✅ Не требует транзакции: встроенные документы
// orders: {
//   _id,
//   items: [
//     { product: "laptop", price: 1000 },
//     { product: "mouse", price: 50 }
//   ]
// }
```

---

## Когда использовать транзакции

### Используйте транзакции для:

1. **Финансовые операции** - переводы, платежи, балансы
2. **Резервирование ресурсов** - товары, билеты, места
3. **Связанные обновления** - когда несколько документов должны быть согласованы
4. **Saga паттерн** - многошаговые бизнес-процессы

### Избегайте транзакций когда:

1. **Достаточно однодокументной операции** - используйте встроенные документы
2. **Eventual consistency приемлема** - асинхронные обновления
3. **Высокая нагрузка** - транзакции добавляют overhead
4. **Операции не связаны логически** - batching не требует транзакций

---

## Итоги

| Аспект | Рекомендация |
|--------|--------------|
| API | Используйте `withTransaction()` с callback |
| Read Concern | `snapshot` для транзакций |
| Write Concern | `majority` для надёжности |
| Время выполнения | Минимизируйте, избегайте внешних вызовов |
| Проектирование | Предпочитайте embedded documents |
| Retry | Автоматически с `withTransaction()` |
| Мониторинг | Следите за метриками и slowms |

Транзакции в MongoDB — мощный инструмент, но используйте их осознанно. Правильное проектирование схемы данных часто позволяет избежать необходимости в многодокументных транзакциях.
