# ethers.js

## Обзор

**ethers.js** — это компактная и полнофункциональная JavaScript-библиотека для взаимодействия с блокчейном Ethereum и совместимыми сетями. Разработана Ричардом Муром (Richard Moore) как альтернатива web3.js с акцентом на безопасность, компактность и простоту использования.

### Ключевые характеристики

- **Размер**: ~120 КБ (минифицированный), ~35 КБ (gzipped)
- **Лицензия**: MIT
- **Версия**: v6.x (актуальная), v5.x (legacy)
- **TypeScript**: Полная поддержка из коробки
- **Модульность**: Можно импортировать только нужные модули

## Архитектура

### Основные модули

```
ethers
├── providers      # Подключение к сети
├── signers        # Управление ключами и подписями
├── contracts      # Взаимодействие со смарт-контрактами
├── utils          # Утилиты (форматирование, хеширование)
├── wallet         # Кошельки
└── abi            # Кодирование/декодирование ABI
```

### Провайдеры (Providers)

Провайдеры обеспечивают только чтение данных из блокчейна:

```javascript
import { ethers } from "ethers";

// JSON-RPC провайдер
const provider = new ethers.JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");

// Провайдер браузерного кошелька (MetaMask)
const browserProvider = new ethers.BrowserProvider(window.ethereum);

// Публичный провайдер (для тестирования)
const defaultProvider = ethers.getDefaultProvider("mainnet");

// WebSocket провайдер для подписок
const wsProvider = new ethers.WebSocketProvider("wss://mainnet.infura.io/ws/v3/YOUR_KEY");
```

### Подписанты (Signers)

Signers могут подписывать транзакции и сообщения:

```javascript
// Создание кошелька из приватного ключа
const wallet = new ethers.Wallet(privateKey, provider);

// Создание случайного кошелька
const randomWallet = ethers.Wallet.createRandom();

// Получение signer из браузерного провайдера
const signer = await browserProvider.getSigner();
```

## Установка

```bash
# npm
npm install ethers

# yarn
yarn add ethers

# pnpm
pnpm add ethers
```

### Импорт

```javascript
// ES6 модули
import { ethers } from "ethers";

// CommonJS
const { ethers } = require("ethers");

// Выборочный импорт (для оптимизации бандла)
import { JsonRpcProvider, Contract, formatEther } from "ethers";
```

## Основные операции

### Работа с балансами

```javascript
import { ethers, formatEther, parseEther } from "ethers";

const provider = new ethers.JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");

// Получение баланса
const balance = await provider.getBalance("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d");
console.log("Баланс в wei:", balance.toString());
console.log("Баланс в ETH:", formatEther(balance));

// Конвертация ETH в wei
const weiAmount = parseEther("1.5");
console.log("1.5 ETH в wei:", weiAmount.toString());
```

### Работа с блоками

```javascript
// Получение текущего номера блока
const blockNumber = await provider.getBlockNumber();
console.log("Текущий блок:", blockNumber);

// Получение блока
const block = await provider.getBlock(blockNumber);
console.log("Хеш блока:", block.hash);
console.log("Timestamp:", new Date(block.timestamp * 1000));
console.log("Количество транзакций:", block.transactions.length);

// Получение блока с транзакциями
const blockWithTxs = await provider.getBlock(blockNumber, true);
```

### Работа с транзакциями

```javascript
// Получение транзакции
const tx = await provider.getTransaction("0x...");
console.log("От:", tx.from);
console.log("Кому:", tx.to);
console.log("Значение:", formatEther(tx.value));
console.log("Gas Limit:", tx.gasLimit.toString());

// Получение receipt (после подтверждения)
const receipt = await provider.getTransactionReceipt("0x...");
console.log("Статус:", receipt.status === 1 ? "Успешно" : "Неудачно");
console.log("Использовано газа:", receipt.gasUsed.toString());
console.log("Номер блока:", receipt.blockNumber);
```

### Отправка транзакций

```javascript
import { ethers, parseEther } from "ethers";

const provider = new ethers.JsonRpcProvider("https://sepolia.infura.io/v3/YOUR_KEY");
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// Простой перевод ETH
const tx = await wallet.sendTransaction({
  to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d",
  value: parseEther("0.1")
});

console.log("Хеш транзакции:", tx.hash);

// Ожидание подтверждения
const receipt = await tx.wait();
console.log("Транзакция подтверждена в блоке:", receipt.blockNumber);

// Транзакция с параметрами газа (EIP-1559)
const txWithGas = await wallet.sendTransaction({
  to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d",
  value: parseEther("0.1"),
  maxFeePerGas: ethers.parseUnits("50", "gwei"),
  maxPriorityFeePerGas: ethers.parseUnits("2", "gwei"),
  gasLimit: 21000
});
```

## Работа со смарт-контрактами

### Подключение к контракту

```javascript
import { ethers, Contract } from "ethers";

// ABI контракта (можно получить из Etherscan или при компиляции)
const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function totalSupply() view returns (uint256)",
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "event Transfer(address indexed from, address indexed to, uint256 value)",
  "event Approval(address indexed owner, address indexed spender, uint256 value)"
];

const provider = new ethers.JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");

// Адрес USDT
const USDT_ADDRESS = "0xdAC17F958D2ee523a2206206994597C13D831ec7";

// Создание экземпляра контракта (только чтение)
const usdtContract = new Contract(USDT_ADDRESS, ERC20_ABI, provider);
```

### Чтение данных контракта

```javascript
// Получение информации о токене
const name = await usdtContract.name();
const symbol = await usdtContract.symbol();
const decimals = await usdtContract.decimals();
const totalSupply = await usdtContract.totalSupply();

console.log(`${name} (${symbol})`);
console.log("Decimals:", decimals);
console.log("Total Supply:", ethers.formatUnits(totalSupply, decimals));

// Получение баланса
const balance = await usdtContract.balanceOf("0x...");
console.log("Баланс:", ethers.formatUnits(balance, decimals), symbol);
```

### Запись в контракт

```javascript
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// Подключение контракта к signer для записи
const usdtWithSigner = usdtContract.connect(wallet);

// Отправка токенов
const tx = await usdtWithSigner.transfer(
  "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d",
  ethers.parseUnits("100", 6) // USDT имеет 6 decimals
);

console.log("TX Hash:", tx.hash);
const receipt = await tx.wait();
console.log("Подтверждено в блоке:", receipt.blockNumber);

// Approve для DeFi протокола
const approveTx = await usdtWithSigner.approve(
  "0xRouterAddress...",
  ethers.MaxUint256 // Безлимитный approve
);
await approveTx.wait();
```

### Работа с событиями

```javascript
// Подписка на события
usdtContract.on("Transfer", (from, to, value, event) => {
  console.log(`Transfer: ${from} -> ${to}: ${ethers.formatUnits(value, 6)} USDT`);
  console.log("Block:", event.log.blockNumber);
});

// Получение исторических событий
const filter = usdtContract.filters.Transfer(
  "0x...", // from (null для любого)
  null     // to (null для любого)
);

const events = await usdtContract.queryFilter(filter, -10000); // последние 10000 блоков

events.forEach(event => {
  console.log(`Block ${event.blockNumber}: ${event.args.from} -> ${event.args.to}`);
});

// Отписка от событий
usdtContract.removeAllListeners("Transfer");
```

## Работа с кошельками

### Создание и импорт кошельков

```javascript
import { ethers, Wallet, HDNodeWallet } from "ethers";

// Создание случайного кошелька
const randomWallet = Wallet.createRandom();
console.log("Адрес:", randomWallet.address);
console.log("Приватный ключ:", randomWallet.privateKey);
console.log("Мнемоника:", randomWallet.mnemonic.phrase);

// Импорт из приватного ключа
const walletFromKey = new Wallet("0x...");

// Импорт из мнемоники
const walletFromMnemonic = Wallet.fromPhrase(
  "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"
);

// HD кошелёк с кастомным путём деривации
const hdNode = HDNodeWallet.fromPhrase(mnemonic);
const child0 = hdNode.derivePath("m/44'/60'/0'/0/0");
const child1 = hdNode.derivePath("m/44'/60'/0'/0/1");
```

### Шифрование кошелька

```javascript
// Шифрование в JSON keystore
const encryptedJson = await wallet.encrypt("password123");

// Сохранение в файл (Node.js)
const fs = require("fs");
fs.writeFileSync("./keystore.json", encryptedJson);

// Расшифровка
const decryptedWallet = await Wallet.fromEncryptedJson(encryptedJson, "password123");
```

### Подпись сообщений

```javascript
const wallet = new Wallet(PRIVATE_KEY);

// Подпись сообщения
const message = "Hello, Ethereum!";
const signature = await wallet.signMessage(message);
console.log("Подпись:", signature);

// Верификация подписи
const recoveredAddress = ethers.verifyMessage(message, signature);
console.log("Восстановленный адрес:", recoveredAddress);
console.log("Совпадает:", recoveredAddress === wallet.address);

// Подпись типизированных данных (EIP-712)
const domain = {
  name: "MyApp",
  version: "1",
  chainId: 1,
  verifyingContract: "0x..."
};

const types = {
  Person: [
    { name: "name", type: "string" },
    { name: "wallet", type: "address" }
  ]
};

const value = {
  name: "Alice",
  wallet: "0x..."
};

const typedSignature = await wallet.signTypedData(domain, types, value);
```

## Утилиты

### Форматирование и парсинг

```javascript
import { ethers } from "ethers";

// ETH <-> Wei
ethers.parseEther("1.5");        // 1500000000000000000n
ethers.formatEther(1500000000000000000n); // "1.5"

// Произвольные единицы
ethers.parseUnits("100", 6);     // 100000000n (USDT decimals)
ethers.formatUnits(100000000n, 6); // "100.0"

// Gwei
ethers.parseUnits("50", "gwei"); // 50000000000n
ethers.formatUnits(50000000000n, "gwei"); // "50.0"
```

### Хеширование

```javascript
// Keccak256
const hash = ethers.keccak256(ethers.toUtf8Bytes("Hello"));
console.log(hash);

// Сигнатура функции
const selector = ethers.id("transfer(address,uint256)").slice(0, 10);
console.log(selector); // "0xa9059cbb"

// SHA256
const sha256Hash = ethers.sha256(ethers.toUtf8Bytes("Hello"));
```

### Работа с адресами

```javascript
// Проверка валидности
ethers.isAddress("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"); // true
ethers.isAddress("invalid"); // false

// Checksummed адрес
ethers.getAddress("0x742d35cc6634c0532925a3b844bc9e7595f0ab3d");
// "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"

// Вычисление адреса контракта
const contractAddress = ethers.getCreateAddress({
  from: "0x...",
  nonce: 5
});
```

### ABI Coder

```javascript
import { AbiCoder } from "ethers";

const abiCoder = new AbiCoder();

// Кодирование
const encoded = abiCoder.encode(
  ["address", "uint256"],
  ["0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d", 1000]
);

// Декодирование
const decoded = abiCoder.decode(
  ["address", "uint256"],
  encoded
);
console.log(decoded[0], decoded[1].toString());
```

## Продвинутые возможности

### Мультиколл (Batch Requests)

```javascript
// Параллельные запросы
const [balance1, balance2, blockNumber] = await Promise.all([
  provider.getBalance("0x..."),
  provider.getBalance("0x..."),
  provider.getBlockNumber()
]);

// Использование Multicall контракта
const MULTICALL_ABI = [
  "function aggregate(tuple(address target, bytes callData)[] calls) returns (uint256 blockNumber, bytes[] returnData)"
];

const multicall = new Contract(MULTICALL_ADDRESS, MULTICALL_ABI, provider);

const calls = [
  {
    target: TOKEN_ADDRESS,
    callData: usdtContract.interface.encodeFunctionData("balanceOf", [addr1])
  },
  {
    target: TOKEN_ADDRESS,
    callData: usdtContract.interface.encodeFunctionData("balanceOf", [addr2])
  }
];

const [, results] = await multicall.aggregate(calls);
```

### Оценка газа

```javascript
// Оценка газа для транзакции
const estimatedGas = await usdtWithSigner.transfer.estimateGas(
  "0x...",
  ethers.parseUnits("100", 6)
);
console.log("Оценка газа:", estimatedGas.toString());

// Получение цены газа
const feeData = await provider.getFeeData();
console.log("Gas Price:", ethers.formatUnits(feeData.gasPrice, "gwei"), "gwei");
console.log("Max Fee:", ethers.formatUnits(feeData.maxFeePerGas, "gwei"), "gwei");
console.log("Priority Fee:", ethers.formatUnits(feeData.maxPriorityFeePerGas, "gwei"), "gwei");
```

### ENS (Ethereum Name Service)

```javascript
// Резолв ENS имени в адрес
const address = await provider.resolveName("vitalik.eth");
console.log(address);

// Обратный резолв
const ensName = await provider.lookupAddress("0x...");
console.log(ensName);

// Получение аватара
const avatar = await provider.getAvatar("vitalik.eth");
```

## Обработка ошибок

```javascript
import { ethers } from "ethers";

try {
  const tx = await wallet.sendTransaction({
    to: "0x...",
    value: ethers.parseEther("100") // Больше, чем есть
  });
  await tx.wait();
} catch (error) {
  if (error.code === "INSUFFICIENT_FUNDS") {
    console.error("Недостаточно средств");
  } else if (error.code === "NONCE_EXPIRED") {
    console.error("Nonce устарел");
  } else if (error.code === "REPLACEMENT_UNDERPRICED") {
    console.error("Цена замены слишком низкая");
  } else if (error.code === "CALL_EXCEPTION") {
    console.error("Контракт отклонил вызов:", error.reason);
  } else if (error.code === "NETWORK_ERROR") {
    console.error("Ошибка сети");
  } else {
    console.error("Неизвестная ошибка:", error);
  }
}
```

## Сравнение с другими библиотеками

| Характеристика | ethers.js | web3.js | viem |
|---------------|-----------|---------|------|
| Размер | ~120 КБ | ~500 КБ | ~35 КБ |
| TypeScript | Отлично | Хорошо | Отлично |
| Модульность | Высокая | Средняя | Высокая |
| Tree-shaking | Да | Ограничено | Да |
| ENS поддержка | Встроена | Плагин | Встроена |
| Документация | Отличная | Хорошая | Отличная |
| Сообщество | Большое | Очень большое | Растущее |

### Преимущества ethers.js

1. **Компактность** — значительно меньше web3.js
2. **Чистая архитектура** — разделение Provider/Signer
3. **Human-readable ABI** — упрощённый синтаксис ABI
4. **Отличная документация** — подробные примеры
5. **Встроенный ENS** — без дополнительных плагинов

### Недостатки

1. **Миграция v5 -> v6** — значительные изменения API
2. **BigInt вместо BigNumber** — в v6, требует адаптации
3. **Менее популярен** — чем web3.js в legacy проектах

## Best Practices

### 1. Безопасность

```javascript
// Никогда не хардкодьте приватные ключи
const privateKey = process.env.PRIVATE_KEY;

// Используйте отдельные кошельки для тестирования
const isMainnet = chainId === 1n;
if (isMainnet && !process.env.PRODUCTION) {
  throw new Error("Mainnet заблокирован в dev окружении");
}
```

### 2. Управление провайдерами

```javascript
// Используйте fallback провайдеры
const provider = new ethers.FallbackProvider([
  new ethers.JsonRpcProvider("https://mainnet.infura.io/v3/KEY"),
  new ethers.JsonRpcProvider("https://eth-mainnet.alchemyapi.io/v2/KEY"),
  new ethers.JsonRpcProvider("https://rpc.ankr.com/eth")
]);

// Переподключение при ошибках
provider.on("error", (error) => {
  console.error("Provider error:", error);
  // Логика переподключения
});
```

### 3. Оптимизация газа

```javascript
// Всегда оценивайте газ перед отправкой
const estimatedGas = await contract.method.estimateGas(args);
const gasLimit = estimatedGas * 120n / 100n; // +20% buffer

const tx = await contract.method(args, { gasLimit });
```

### 4. Обработка pending транзакций

```javascript
// Ожидание с таймаутом
const tx = await wallet.sendTransaction(txData);

const receipt = await Promise.race([
  tx.wait(),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), 60000)
  )
]);
```

## Ресурсы

- [Официальная документация](https://docs.ethers.org/v6/)
- [GitHub репозиторий](https://github.com/ethers-io/ethers.js)
- [Changelog v5 -> v6](https://docs.ethers.org/v6/migrating/)
- [Ethers Playground](https://playground.ethers.org/)
