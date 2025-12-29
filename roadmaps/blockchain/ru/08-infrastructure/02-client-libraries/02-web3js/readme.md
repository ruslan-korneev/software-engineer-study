# web3.js

## Обзор

**web3.js** — это официальная JavaScript-библиотека для взаимодействия с Ethereum, разрабатываемая Ethereum Foundation. Это старейшая и наиболее распространённая библиотека в экосистеме, ставшая стандартом де-факто для Web3 разработки.

### Ключевые характеристики

- **Размер**: ~500 КБ (минифицированный)
- **Лицензия**: LGPL-3.0
- **Версия**: v4.x (актуальная), v1.x (legacy)
- **TypeScript**: Полная поддержка в v4
- **История**: С 2015 года, наиболее зрелая библиотека

## Архитектура

### Модульная структура (v4.x)

```
web3
├── web3-eth           # Основной модуль Ethereum
├── web3-eth-accounts  # Управление аккаунтами
├── web3-eth-contract  # Работа с контрактами
├── web3-eth-abi       # ABI кодирование/декодирование
├── web3-eth-ens       # Ethereum Name Service
├── web3-net           # Сетевые утилиты
├── web3-utils         # Вспомогательные функции
├── web3-providers-*   # Различные провайдеры
└── web3-validator     # Валидация данных
```

### Сравнение v1 и v4

| Аспект | v1.x | v4.x |
|--------|------|------|
| TypeScript | Частичный | Полный |
| Модульность | Монолит | Отдельные пакеты |
| BigInt | BN.js | Нативный BigInt |
| Tree-shaking | Нет | Да |
| ESM | Нет | Да |

## Установка

```bash
# Полная установка
npm install web3

# Установка отдельных модулей (v4)
npm install web3-eth web3-eth-contract web3-utils

# Yarn
yarn add web3

# pnpm
pnpm add web3
```

### Импорт

```javascript
// ES6 модули (v4)
import { Web3 } from "web3";
import { Contract } from "web3-eth-contract";

// CommonJS
const { Web3 } = require("web3");

// Выборочный импорт
import { toWei, fromWei, isAddress } from "web3-utils";
```

## Инициализация

### Подключение к сети

```javascript
import { Web3 } from "web3";

// HTTP провайдер
const web3 = new Web3("https://mainnet.infura.io/v3/YOUR_KEY");

// WebSocket провайдер
const web3Ws = new Web3("wss://mainnet.infura.io/ws/v3/YOUR_KEY");

// Инъекция провайдера (MetaMask)
const web3Browser = new Web3(window.ethereum);

// IPC провайдер (локальная нода)
const web3Ipc = new Web3(new Web3.providers.IpcProvider("/path/to/geth.ipc"));

// Без провайдера (только утилиты)
const web3Utils = new Web3();
```

### Проверка подключения

```javascript
// Проверка версии ноды
const nodeInfo = await web3.eth.getNodeInfo();
console.log("Нода:", nodeInfo);

// ID сети
const chainId = await web3.eth.getChainId();
console.log("Chain ID:", chainId);

// Синхронизация
const isSyncing = await web3.eth.isSyncing();
console.log("Синхронизация:", isSyncing);

// Проверка подключения
const isConnected = await web3.eth.net.isListening();
console.log("Подключено:", isConnected);
```

## Основные операции

### Работа с аккаунтами

```javascript
// Получение списка аккаунтов (если нода управляет ключами)
const accounts = await web3.eth.getAccounts();

// Получение баланса
const balance = await web3.eth.getBalance("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d");
console.log("Баланс в wei:", balance);
console.log("Баланс в ETH:", web3.utils.fromWei(balance, "ether"));

// Получение nonce
const nonce = await web3.eth.getTransactionCount("0x...");
console.log("Transaction count:", nonce);
```

### Работа с блоками

```javascript
// Текущий номер блока
const blockNumber = await web3.eth.getBlockNumber();
console.log("Текущий блок:", blockNumber);

// Получение блока
const block = await web3.eth.getBlock(blockNumber);
console.log("Блок:", block);

// Блок с полными данными транзакций
const blockWithTxs = await web3.eth.getBlock(blockNumber, true);
console.log("Транзакции:", blockWithTxs.transactions);

// Получение блока по хешу
const blockByHash = await web3.eth.getBlock("0x...");

// Количество транзакций в блоке
const txCount = await web3.eth.getBlockTransactionCount(blockNumber);
```

### Работа с транзакциями

```javascript
// Получение транзакции по хешу
const tx = await web3.eth.getTransaction("0x...");
console.log("Транзакция:", tx);

// Получение receipt
const receipt = await web3.eth.getTransactionReceipt("0x...");
console.log("Статус:", receipt.status);
console.log("Газ использовано:", receipt.gasUsed);

// Оценка газа
const gasEstimate = await web3.eth.estimateGas({
  from: "0x...",
  to: "0x...",
  value: web3.utils.toWei("1", "ether")
});
console.log("Оценка газа:", gasEstimate);

// Текущая цена газа
const gasPrice = await web3.eth.getGasPrice();
console.log("Gas price:", web3.utils.fromWei(gasPrice, "gwei"), "gwei");
```

### Отправка транзакций

```javascript
// Простой перевод ETH
const tx = await web3.eth.sendTransaction({
  from: accounts[0],
  to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d",
  value: web3.utils.toWei("0.1", "ether"),
  gas: 21000
});

console.log("TX Hash:", tx.transactionHash);
console.log("Block:", tx.blockNumber);

// С параметрами EIP-1559
const eip1559Tx = await web3.eth.sendTransaction({
  from: accounts[0],
  to: "0x...",
  value: web3.utils.toWei("0.1", "ether"),
  maxFeePerGas: web3.utils.toWei("50", "gwei"),
  maxPriorityFeePerGas: web3.utils.toWei("2", "gwei")
});
```

## Управление аккаунтами

### Создание аккаунтов

```javascript
import { Web3 } from "web3";

const web3 = new Web3();

// Создание нового аккаунта
const account = web3.eth.accounts.create();
console.log("Адрес:", account.address);
console.log("Приватный ключ:", account.privateKey);

// Создание из приватного ключа
const importedAccount = web3.eth.accounts.privateKeyToAccount("0x...");

// Создание из энтропии
const accountWithEntropy = web3.eth.accounts.create("дополнительная энтропия");
```

### Подпись транзакций

```javascript
// Подпись транзакции
const signedTx = await web3.eth.accounts.signTransaction({
  to: "0x...",
  value: web3.utils.toWei("1", "ether"),
  gas: 21000,
  gasPrice: await web3.eth.getGasPrice(),
  nonce: await web3.eth.getTransactionCount(account.address)
}, account.privateKey);

console.log("Подписанная транзакция:", signedTx.rawTransaction);

// Отправка подписанной транзакции
const receipt = await web3.eth.sendSignedTransaction(signedTx.rawTransaction);
console.log("Receipt:", receipt);
```

### Подпись сообщений

```javascript
// Подпись сообщения
const message = "Hello, Ethereum!";
const signature = web3.eth.accounts.sign(message, privateKey);
console.log("Подпись:", signature.signature);

// Восстановление адреса
const recoveredAddress = web3.eth.accounts.recover(message, signature.signature);
console.log("Восстановленный адрес:", recoveredAddress);

// Подпись с помощью injected provider (MetaMask)
const signatureMetaMask = await web3.eth.personal.sign(message, accounts[0], "");
```

### Keystore (шифрование)

```javascript
// Шифрование приватного ключа
const encrypted = await web3.eth.accounts.encrypt(privateKey, "password123");
console.log("Зашифрованный keystore:", JSON.stringify(encrypted));

// Расшифровка
const decrypted = await web3.eth.accounts.decrypt(encrypted, "password123");
console.log("Расшифрованный аккаунт:", decrypted.address);
```

## Работа со смарт-контрактами

### Создание экземпляра контракта

```javascript
import { Web3 } from "web3";

const web3 = new Web3("https://mainnet.infura.io/v3/YOUR_KEY");

// ABI контракта
const ERC20_ABI = [
  {
    "constant": true,
    "inputs": [],
    "name": "name",
    "outputs": [{"name": "", "type": "string"}],
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "symbol",
    "outputs": [{"name": "", "type": "string"}],
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "decimals",
    "outputs": [{"name": "", "type": "uint8"}],
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [{"name": "owner", "type": "address"}],
    "name": "balanceOf",
    "outputs": [{"name": "", "type": "uint256"}],
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {"name": "to", "type": "address"},
      {"name": "value", "type": "uint256"}
    ],
    "name": "transfer",
    "outputs": [{"name": "", "type": "bool"}],
    "type": "function"
  },
  {
    "anonymous": false,
    "inputs": [
      {"indexed": true, "name": "from", "type": "address"},
      {"indexed": true, "name": "to", "type": "address"},
      {"indexed": false, "name": "value", "type": "uint256"}
    ],
    "name": "Transfer",
    "type": "event"
  }
];

const USDT_ADDRESS = "0xdAC17F958D2ee523a2206206994597C13D831ec7";

// Создание экземпляра контракта
const usdtContract = new web3.eth.Contract(ERC20_ABI, USDT_ADDRESS);
```

### Чтение данных (call)

```javascript
// Вызов view/pure функций
const name = await usdtContract.methods.name().call();
const symbol = await usdtContract.methods.symbol().call();
const decimals = await usdtContract.methods.decimals().call();

console.log(`Токен: ${name} (${symbol})`);
console.log("Decimals:", decimals);

// Получение баланса
const balance = await usdtContract.methods.balanceOf("0x...").call();
console.log("Баланс:", balance / 10n ** BigInt(decimals), symbol);

// Вызов с определённого адреса (для msg.sender)
const result = await contract.methods.someMethod().call({ from: "0x..." });
```

### Запись данных (send)

```javascript
// Добавление аккаунта в wallet
web3.eth.accounts.wallet.add(privateKey);

// Отправка транзакции
const tx = await usdtContract.methods
  .transfer("0x...", 1000000n) // 1 USDT (6 decimals)
  .send({
    from: "0x...",
    gas: 100000
  });

console.log("TX Hash:", tx.transactionHash);

// С обработкой событий
usdtContract.methods
  .transfer("0x...", 1000000n)
  .send({ from: "0x...", gas: 100000 })
  .on("transactionHash", (hash) => {
    console.log("TX Hash:", hash);
  })
  .on("confirmation", (confirmationNumber, receipt) => {
    console.log("Подтверждение:", confirmationNumber);
  })
  .on("receipt", (receipt) => {
    console.log("Receipt получен");
  })
  .on("error", (error) => {
    console.error("Ошибка:", error);
  });
```

### Оценка газа для контрактных вызовов

```javascript
const gasEstimate = await usdtContract.methods
  .transfer("0x...", 1000000n)
  .estimateGas({ from: "0x..." });

console.log("Оценка газа:", gasEstimate);
```

### Работа с событиями

```javascript
// Подписка на события (WebSocket)
const subscription = usdtContract.events.Transfer({
  filter: {
    from: "0x..." // Опциональный фильтр
  }
})
.on("connected", (subscriptionId) => {
  console.log("Подписка создана:", subscriptionId);
})
.on("data", (event) => {
  console.log("Transfer:", {
    from: event.returnValues.from,
    to: event.returnValues.to,
    value: event.returnValues.value
  });
})
.on("error", (error) => {
  console.error("Ошибка подписки:", error);
});

// Отписка
subscription.unsubscribe();

// Получение прошлых событий
const events = await usdtContract.getPastEvents("Transfer", {
  filter: { from: "0x..." },
  fromBlock: 18000000,
  toBlock: "latest"
});

events.forEach(event => {
  console.log(`Block ${event.blockNumber}: ${event.returnValues.value}`);
});
```

## Утилиты (web3-utils)

### Конвертация единиц

```javascript
import { Web3 } from "web3";
const web3 = new Web3();

// ETH <-> Wei
web3.utils.toWei("1", "ether");     // "1000000000000000000"
web3.utils.toWei("1.5", "ether");   // "1500000000000000000"
web3.utils.fromWei("1000000000000000000", "ether"); // "1"

// Другие единицы
web3.utils.toWei("50", "gwei");     // "50000000000"
web3.utils.fromWei("50000000000", "gwei"); // "50"

// Единицы измерения:
// wei, kwei, mwei, gwei, szabo, finney, ether
```

### Работа с адресами

```javascript
// Проверка адреса
web3.utils.isAddress("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"); // true
web3.utils.isAddress("invalid"); // false

// Checksum адрес
web3.utils.toChecksumAddress("0x742d35cc6634c0532925a3b844bc9e7595f0ab3d");
// "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"

// Проверка checksum
web3.utils.checkAddressChecksum("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"); // true
```

### Хеширование

```javascript
// Keccak256 (SHA3)
web3.utils.keccak256("Hello");
// или
web3.utils.sha3("Hello");

// Для текста
web3.utils.soliditySha3("Hello");
web3.utils.soliditySha3({ type: "string", value: "Hello" });

// Несколько аргументов (как в Solidity)
web3.utils.soliditySha3(
  { type: "address", value: "0x..." },
  { type: "uint256", value: 1000 }
);

// SHA256
web3.utils.sha256("Hello");
```

### Кодирование данных

```javascript
// HEX <-> строки
web3.utils.toHex("Hello");           // "0x48656c6c6f"
web3.utils.hexToUtf8("0x48656c6c6f"); // "Hello"
web3.utils.utf8ToHex("Hello");       // "0x48656c6c6f"

// HEX <-> числа
web3.utils.toHex(256);               // "0x100"
web3.utils.hexToNumber("0x100");     // 256n
web3.utils.numberToHex(256);         // "0x100"

// Проверка типов
web3.utils.isHexStrict("0x1234");    // true
web3.utils.isHex("1234");            // true

// Padding
web3.utils.padLeft("0x3456", 10);    // "0x0000003456"
web3.utils.padRight("0x3456", 10);   // "0x3456000000"
```

### Работа с числами

```javascript
// Генерация случайного hex
web3.utils.randomHex(32); // 32 байта случайных данных

// BN (BigNumber) - для v1.x
const BN = web3.utils.BN;
const a = new BN("1000000000000000000");
const b = new BN("2000000000000000000");
const sum = a.add(b);

// В v4.x используется нативный BigInt
const aBigInt = 1000000000000000000n;
const bBigInt = 2000000000000000000n;
const sumBigInt = aBigInt + bBigInt;
```

## ABI кодирование

```javascript
import { Web3 } from "web3";
const web3 = new Web3();

// Кодирование параметров функции
const encoded = web3.eth.abi.encodeParameters(
  ["address", "uint256"],
  ["0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d", 1000]
);

// Декодирование
const decoded = web3.eth.abi.decodeParameters(
  ["address", "uint256"],
  encoded
);

// Кодирование вызова функции
const functionCall = web3.eth.abi.encodeFunctionCall({
  name: "transfer",
  type: "function",
  inputs: [
    { type: "address", name: "to" },
    { type: "uint256", name: "value" }
  ]
}, ["0x...", "1000"]);

// Кодирование сигнатуры функции
const signature = web3.eth.abi.encodeFunctionSignature("transfer(address,uint256)");
// "0xa9059cbb"

// Кодирование сигнатуры события
const eventSignature = web3.eth.abi.encodeEventSignature(
  "Transfer(address,address,uint256)"
);
```

## Подписки и WebSocket

```javascript
import { Web3 } from "web3";

const web3 = new Web3("wss://mainnet.infura.io/ws/v3/YOUR_KEY");

// Подписка на новые блоки
const blockSubscription = await web3.eth.subscribe("newBlockHeaders");

blockSubscription.on("data", (block) => {
  console.log("Новый блок:", block.number);
});

blockSubscription.on("error", (error) => {
  console.error("Ошибка:", error);
});

// Подписка на pending транзакции
const pendingSubscription = await web3.eth.subscribe("pendingTransactions");

pendingSubscription.on("data", (txHash) => {
  console.log("Pending TX:", txHash);
});

// Подписка на логи
const logsSubscription = await web3.eth.subscribe("logs", {
  address: "0x...",
  topics: [web3.utils.sha3("Transfer(address,address,uint256)")]
});

logsSubscription.on("data", (log) => {
  console.log("Log:", log);
});

// Отписка
await blockSubscription.unsubscribe();
```

## ENS (Ethereum Name Service)

```javascript
import { Web3 } from "web3";

const web3 = new Web3("https://mainnet.infura.io/v3/YOUR_KEY");

// Резолв ENS в адрес
const address = await web3.eth.ens.getAddress("vitalik.eth");
console.log("Адрес:", address);

// Обратный резолв
const name = await web3.eth.ens.getName("0x...");
console.log("ENS имя:", name);

// Получение владельца
const owner = await web3.eth.ens.getOwner("vitalik.eth");
console.log("Владелец:", owner);

// Получение resolver
const resolver = await web3.eth.ens.getResolver("vitalik.eth");
console.log("Resolver:", resolver);
```

## Batch запросы

```javascript
// Создание batch
const batch = new web3.eth.BatchRequest();

// Добавление запросов
const balanceRequest = batch.add(
  web3.eth.getBalance.request("0x...", "latest")
);
const blockRequest = batch.add(
  web3.eth.getBlockNumber.request()
);
const txCountRequest = batch.add(
  web3.eth.getTransactionCount.request("0x...", "latest")
);

// Выполнение batch
const results = await batch.execute();

// Получение результатов
const balance = await balanceRequest;
const blockNumber = await blockRequest;
const txCount = await txCountRequest;
```

## Обработка ошибок

```javascript
import { Web3 } from "web3";

const web3 = new Web3("https://mainnet.infura.io/v3/YOUR_KEY");

try {
  const tx = await web3.eth.sendTransaction({
    from: "0x...",
    to: "0x...",
    value: web3.utils.toWei("100", "ether")
  });
} catch (error) {
  if (error.message.includes("insufficient funds")) {
    console.error("Недостаточно средств");
  } else if (error.message.includes("nonce too low")) {
    console.error("Nonce слишком низкий");
  } else if (error.message.includes("gas required exceeds")) {
    console.error("Недостаточно газа");
  } else if (error.message.includes("execution reverted")) {
    console.error("Контракт отклонил транзакцию");
    // Попытка получить причину
    const reason = error.data?.message || error.reason;
    console.error("Причина:", reason);
  } else {
    console.error("Неизвестная ошибка:", error.message);
  }
}

// Для контрактных вызовов
try {
  await contract.methods.someMethod().call();
} catch (error) {
  // Декодирование custom error
  if (error.data) {
    const decodedError = web3.eth.abi.decodeParameter("string", error.data);
    console.error("Custom Error:", decodedError);
  }
}
```

## Плагины (v4)

```javascript
import { Web3, Web3PluginBase } from "web3";

// Создание плагина
class MyPlugin extends Web3PluginBase {
  pluginNamespace = "myPlugin";

  async customMethod() {
    const blockNumber = await this.requestManager.send({
      method: "eth_blockNumber",
      params: []
    });
    return blockNumber;
  }
}

// Регистрация плагина
const web3 = new Web3("https://...");
web3.registerPlugin(new MyPlugin());

// Использование
const result = await web3.myPlugin.customMethod();
```

## Сравнение с другими библиотеками

| Характеристика | web3.js v4 | ethers.js v6 | viem |
|---------------|------------|--------------|------|
| Размер | ~500 КБ | ~120 КБ | ~35 КБ |
| TypeScript | Полный | Отличный | Отличный |
| Tree-shaking | Да | Да | Да |
| Зрелость | Высокая | Высокая | Новая |
| Плагины | Есть | Нет | Нет |
| Сообщество | Очень большое | Большое | Растущее |
| Документация | Хорошая | Отличная | Отличная |

### Преимущества web3.js

1. **Официальная библиотека** — поддержка Ethereum Foundation
2. **Большое сообщество** — много примеров и решений
3. **Плагины** — расширяемость через систему плагинов
4. **Полный набор функций** — всё в одном месте
5. **Совместимость** — работает с большинством инструментов

### Недостатки

1. **Размер** — значительно больше альтернатив
2. **Сложность миграции** — v1 -> v4 требует переписывания
3. **Производительность** — медленнее viem
4. **Историческое наследие** — некоторые устаревшие паттерны

## Best Practices

### 1. Управление провайдерами

```javascript
// Переподключение при ошибках
web3.currentProvider.on("error", (error) => {
  console.error("Provider error:", error);
  // Переподключение
  web3.setProvider(new Web3.providers.WebsocketProvider("wss://..."));
});

// Несколько провайдеров для резервирования
const providers = [
  "https://mainnet.infura.io/v3/KEY1",
  "https://eth-mainnet.alchemyapi.io/v2/KEY2",
  "https://rpc.ankr.com/eth"
];

let currentProviderIndex = 0;

function switchProvider() {
  currentProviderIndex = (currentProviderIndex + 1) % providers.length;
  web3.setProvider(providers[currentProviderIndex]);
}
```

### 2. Безопасность

```javascript
// Проверка входных данных
function validateAddress(address) {
  if (!web3.utils.isAddress(address)) {
    throw new Error("Невалидный адрес");
  }
  return web3.utils.toChecksumAddress(address);
}

// Проверка суммы
function validateAmount(amount) {
  const wei = web3.utils.toWei(amount, "ether");
  if (BigInt(wei) <= 0n) {
    throw new Error("Сумма должна быть положительной");
  }
  return wei;
}
```

### 3. Оптимизация газа

```javascript
// Динамическая оценка газа
async function sendWithOptimalGas(txParams) {
  const gasEstimate = await web3.eth.estimateGas(txParams);
  const gasWithBuffer = gasEstimate * 120n / 100n; // +20%

  const feeData = await web3.eth.calculateFeeData();

  return web3.eth.sendTransaction({
    ...txParams,
    gas: gasWithBuffer,
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas
  });
}
```

### 4. Обработка pending транзакций

```javascript
// Ожидание с подтверждениями
async function sendAndWait(tx, confirmations = 2) {
  const receipt = await tx
    .on("transactionHash", (hash) => {
      console.log("TX отправлена:", hash);
    });

  // Ожидание дополнительных подтверждений
  let currentBlock = await web3.eth.getBlockNumber();
  const targetBlock = receipt.blockNumber + BigInt(confirmations);

  while (currentBlock < targetBlock) {
    await new Promise(resolve => setTimeout(resolve, 12000));
    currentBlock = await web3.eth.getBlockNumber();
  }

  return receipt;
}
```

## Миграция с v1 на v4

```javascript
// v1.x
const Web3 = require("web3");
const web3 = new Web3("https://...");
const balance = await web3.eth.getBalance("0x...");
// balance - строка

// v4.x
import { Web3 } from "web3";
const web3 = new Web3("https://...");
const balance = await web3.eth.getBalance("0x...");
// balance - BigInt

// Изменения в Contract API
// v1.x
contract.methods.transfer(to, amount).send({ from });

// v4.x - тот же синтаксис, но типизированный
contract.methods.transfer(to, amount).send({ from });

// Изменения в utils
// v1.x
web3.utils.toBN("1000000000000000000");

// v4.x - нативный BigInt
BigInt("1000000000000000000");
// или
1000000000000000000n;
```

## Ресурсы

- [Официальная документация](https://docs.web3js.org/)
- [GitHub репозиторий](https://github.com/web3/web3.js)
- [Руководство по миграции v1 -> v4](https://docs.web3js.org/guides/migration_guide/)
- [Примеры кода](https://github.com/web3/web3.js/tree/4.x/packages/web3/examples)
- [Discord сообщество](https://discord.gg/web3js)
