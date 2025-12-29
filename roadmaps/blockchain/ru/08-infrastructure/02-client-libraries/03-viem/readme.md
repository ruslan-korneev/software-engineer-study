# viem

## Обзор

**viem** — это современная TypeScript-библиотека для взаимодействия с Ethereum и EVM-совместимыми сетями. Разработана командой wagmi как легковесная и производительная альтернатива ethers.js и web3.js.

### Ключевые характеристики

- **Размер**: ~35 КБ (минифицированный, gzipped)
- **Лицензия**: MIT
- **TypeScript**: Первый класс, строгая типизация
- **Производительность**: До 15x быстрее ethers.js в некоторых операциях
- **Модульность**: Полный tree-shaking
- **Совместимость**: wagmi, permissionless, и другие современные библиотеки

### Философия viem

1. **Типобезопасность** — максимальное использование TypeScript
2. **Компактность** — минимальный размер бандла
3. **Производительность** — оптимизация критических операций
4. **Композиция** — actions вместо классов
5. **Человекочитаемые ошибки** — понятные сообщения об ошибках

## Архитектура

### Основные концепции

```
viem
├── clients        # Transport + Actions
│   ├── public     # Чтение данных (RPC)
│   ├── wallet     # Подпись и отправка транзакций
│   └── test       # Тестирование (Anvil, Hardhat)
├── actions        # Отдельные операции
├── chains         # Предопределённые сети
├── accounts       # Управление ключами
└── utils          # Вспомогательные функции
```

### Клиенты

Viem использует паттерн клиентов вместо монолитного объекта:

```javascript
import { createPublicClient, createWalletClient, http } from "viem";
import { mainnet } from "viem/chains";

// Public Client — только чтение
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});

// Wallet Client — подпись и отправка
const walletClient = createWalletClient({
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});
```

## Установка

```bash
# npm
npm install viem

# yarn
yarn add viem

# pnpm
pnpm add viem

# bun
bun add viem
```

### Импорт

```typescript
// Основные элементы
import {
  createPublicClient,
  createWalletClient,
  http,
  parseEther,
  formatEther
} from "viem";

// Сети
import { mainnet, sepolia, polygon, arbitrum } from "viem/chains";

// Аккаунты
import { privateKeyToAccount, mnemonicToAccount } from "viem/accounts";

// ABI утилиты
import { parseAbi, parseAbiItem } from "viem";
```

## Транспорты

```typescript
import { http, webSocket, fallback, custom } from "viem";

// HTTP транспорт
const httpTransport = http("https://mainnet.infura.io/v3/YOUR_KEY");

// WebSocket транспорт
const wsTransport = webSocket("wss://mainnet.infura.io/ws/v3/YOUR_KEY");

// Fallback (резервирование)
const fallbackTransport = fallback([
  http("https://mainnet.infura.io/v3/KEY"),
  http("https://eth-mainnet.alchemyapi.io/v2/KEY"),
  http("https://rpc.ankr.com/eth")
]);

// Custom (для браузерных кошельков)
const customTransport = custom(window.ethereum);

// Использование
const client = createPublicClient({
  chain: mainnet,
  transport: fallbackTransport
});
```

## Основные операции

### Работа с балансами

```typescript
import { createPublicClient, http, formatEther, parseEther } from "viem";
import { mainnet } from "viem/chains";

const client = createPublicClient({
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});

// Получение баланса
const balance = await client.getBalance({
  address: "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"
});

console.log("Баланс в wei:", balance);
console.log("Баланс в ETH:", formatEther(balance));

// Парсинг ETH в wei
const weiAmount = parseEther("1.5");
console.log("1.5 ETH в wei:", weiAmount); // 1500000000000000000n
```

### Работа с блоками

```typescript
// Текущий номер блока
const blockNumber = await client.getBlockNumber();
console.log("Текущий блок:", blockNumber);

// Получение блока
const block = await client.getBlock({
  blockNumber: blockNumber
});
console.log("Хеш блока:", block.hash);
console.log("Timestamp:", new Date(Number(block.timestamp) * 1000));

// Блок с транзакциями
const blockWithTxs = await client.getBlock({
  blockNumber: blockNumber,
  includeTransactions: true
});

// Последний блок
const latestBlock = await client.getBlock();

// Подписка на новые блоки
const unwatch = client.watchBlockNumber({
  onBlockNumber: (blockNumber) => {
    console.log("Новый блок:", blockNumber);
  }
});

// Отписка
unwatch();
```

### Работа с транзакциями

```typescript
// Получение транзакции
const tx = await client.getTransaction({
  hash: "0x..."
});
console.log("От:", tx.from);
console.log("Кому:", tx.to);
console.log("Значение:", formatEther(tx.value));

// Получение receipt
const receipt = await client.getTransactionReceipt({
  hash: "0x..."
});
console.log("Статус:", receipt.status); // "success" | "reverted"
console.log("Газ использовано:", receipt.gasUsed);

// Ожидание подтверждения
const confirmedReceipt = await client.waitForTransactionReceipt({
  hash: "0x...",
  confirmations: 2
});

// Количество транзакций (nonce)
const nonce = await client.getTransactionCount({
  address: "0x..."
});
```

### Отправка транзакций

```typescript
import { createWalletClient, http, parseEther } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";

const account = privateKeyToAccount("0x...");

const walletClient = createWalletClient({
  account,
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});

// Отправка ETH
const hash = await walletClient.sendTransaction({
  to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d",
  value: parseEther("0.1")
});

console.log("TX Hash:", hash);

// Ожидание подтверждения (через publicClient)
const receipt = await publicClient.waitForTransactionReceipt({ hash });

// Транзакция с параметрами газа
const hashWithGas = await walletClient.sendTransaction({
  to: "0x...",
  value: parseEther("0.1"),
  maxFeePerGas: parseGwei("50"),
  maxPriorityFeePerGas: parseGwei("2"),
  gas: 21000n
});
```

## Работа с аккаунтами

### Локальные аккаунты

```typescript
import {
  privateKeyToAccount,
  mnemonicToAccount,
  generatePrivateKey,
  generateMnemonic
} from "viem/accounts";

// Генерация нового приватного ключа
const privateKey = generatePrivateKey();

// Создание аккаунта из приватного ключа
const account = privateKeyToAccount(privateKey);
console.log("Адрес:", account.address);

// Создание из мнемоники
const mnemonic = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";
const accountFromMnemonic = mnemonicToAccount(mnemonic);

// С кастомным путём деривации
const accountCustomPath = mnemonicToAccount(mnemonic, {
  path: "m/44'/60'/0'/0/1"
});
```

### JSON-RPC аккаунты (браузерные кошельки)

```typescript
import { createWalletClient, custom } from "viem";
import { mainnet } from "viem/chains";

// Подключение к MetaMask
const walletClient = createWalletClient({
  chain: mainnet,
  transport: custom(window.ethereum)
});

// Запрос аккаунтов
const [address] = await walletClient.requestAddresses();
console.log("Подключённый адрес:", address);

// Получение адресов
const addresses = await walletClient.getAddresses();
```

### Подпись сообщений

```typescript
// Подпись сообщения
const signature = await walletClient.signMessage({
  account,
  message: "Hello, Ethereum!"
});

// Верификация подписи
import { verifyMessage } from "viem";

const valid = await verifyMessage({
  address: account.address,
  message: "Hello, Ethereum!",
  signature
});
console.log("Подпись валидна:", valid);

// Подпись типизированных данных (EIP-712)
const typedSignature = await walletClient.signTypedData({
  account,
  domain: {
    name: "MyApp",
    version: "1",
    chainId: 1,
    verifyingContract: "0x..."
  },
  types: {
    Person: [
      { name: "name", type: "string" },
      { name: "wallet", type: "address" }
    ]
  },
  primaryType: "Person",
  message: {
    name: "Alice",
    wallet: "0x..."
  }
});
```

## Работа со смарт-контрактами

### Чтение данных контракта

```typescript
import { createPublicClient, http, parseAbi } from "viem";
import { mainnet } from "viem/chains";

const client = createPublicClient({
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});

// Human-readable ABI
const abi = parseAbi([
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 value)"
]);

const USDT_ADDRESS = "0xdAC17F958D2ee523a2206206994597C13D831ec7";

// Одиночный вызов
const name = await client.readContract({
  address: USDT_ADDRESS,
  abi,
  functionName: "name"
});
console.log("Имя токена:", name);

// Вызов с аргументами
const balance = await client.readContract({
  address: USDT_ADDRESS,
  abi,
  functionName: "balanceOf",
  args: ["0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"]
});
console.log("Баланс:", balance);
```

### Мультивызовы (multicall)

```typescript
// Множественные вызовы одним запросом
const results = await client.multicall({
  contracts: [
    {
      address: USDT_ADDRESS,
      abi,
      functionName: "name"
    },
    {
      address: USDT_ADDRESS,
      abi,
      functionName: "symbol"
    },
    {
      address: USDT_ADDRESS,
      abi,
      functionName: "decimals"
    },
    {
      address: USDT_ADDRESS,
      abi,
      functionName: "balanceOf",
      args: ["0x..."]
    }
  ]
});

const [nameResult, symbolResult, decimalsResult, balanceResult] = results;

console.log("Имя:", nameResult.result);
console.log("Символ:", symbolResult.result);
console.log("Decimals:", decimalsResult.result);
console.log("Баланс:", balanceResult.result);
```

### Запись в контракт

```typescript
import { createWalletClient, http, parseAbi, parseUnits } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";

const account = privateKeyToAccount("0x...");

const walletClient = createWalletClient({
  account,
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY")
});

const abi = parseAbi([
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)"
]);

// Отправка токенов
const hash = await walletClient.writeContract({
  address: USDT_ADDRESS,
  abi,
  functionName: "transfer",
  args: ["0x...", parseUnits("100", 6)] // 100 USDT
});

console.log("TX Hash:", hash);

// Ожидание подтверждения
const receipt = await publicClient.waitForTransactionReceipt({ hash });
console.log("Статус:", receipt.status);

// Approve для DeFi
const approveHash = await walletClient.writeContract({
  address: USDT_ADDRESS,
  abi,
  functionName: "approve",
  args: ["0xRouterAddress...", maxUint256]
});
```

### Симуляция вызова

```typescript
// Симуляция перед отправкой
const { request } = await publicClient.simulateContract({
  address: USDT_ADDRESS,
  abi,
  functionName: "transfer",
  args: ["0x...", parseUnits("100", 6)],
  account
});

// Если симуляция успешна, отправляем транзакцию
const hash = await walletClient.writeContract(request);
```

### Оценка газа

```typescript
// Оценка газа для контрактного вызова
const gasEstimate = await publicClient.estimateContractGas({
  address: USDT_ADDRESS,
  abi,
  functionName: "transfer",
  args: ["0x...", parseUnits("100", 6)],
  account: account.address
});

console.log("Оценка газа:", gasEstimate);

// Получение данных о газе
const feeData = await publicClient.estimateFeesPerGas();
console.log("Max Fee:", formatGwei(feeData.maxFeePerGas));
console.log("Priority Fee:", formatGwei(feeData.maxPriorityFeePerGas));
```

## События и логи

### Получение событий

```typescript
// Получение логов
const logs = await publicClient.getLogs({
  address: USDT_ADDRESS,
  event: parseAbiItem("event Transfer(address indexed from, address indexed to, uint256 value)"),
  fromBlock: 18000000n,
  toBlock: "latest"
});

logs.forEach(log => {
  console.log(`Transfer: ${log.args.from} -> ${log.args.to}: ${log.args.value}`);
});

// Логи с фильтрами
const filteredLogs = await publicClient.getLogs({
  address: USDT_ADDRESS,
  event: parseAbiItem("event Transfer(address indexed from, address indexed to, uint256 value)"),
  args: {
    from: "0x..." // Только от этого адреса
  },
  fromBlock: 18000000n,
  toBlock: "latest"
});
```

### Подписка на события

```typescript
// Подписка на события контракта
const unwatch = publicClient.watchContractEvent({
  address: USDT_ADDRESS,
  abi,
  eventName: "Transfer",
  onLogs: (logs) => {
    logs.forEach(log => {
      console.log(`Transfer: ${log.args.from} -> ${log.args.to}`);
    });
  }
});

// Подписка на все события
const unwatchAll = publicClient.watchEvent({
  address: USDT_ADDRESS,
  onLogs: (logs) => {
    console.log("Событие:", logs);
  }
});

// Отписка
unwatch();
```

### Подписка на pending транзакции

```typescript
const unwatch = publicClient.watchPendingTransactions({
  onTransactions: (hashes) => {
    hashes.forEach(hash => {
      console.log("Pending TX:", hash);
    });
  }
});
```

## Утилиты

### Форматирование и парсинг

```typescript
import {
  formatEther, parseEther,
  formatGwei, parseGwei,
  formatUnits, parseUnits
} from "viem";

// ETH <-> Wei
parseEther("1.5");              // 1500000000000000000n
formatEther(1500000000000000000n); // "1.5"

// Gwei
parseGwei("50");                // 50000000000n
formatGwei(50000000000n);       // "50"

// Произвольные единицы
parseUnits("100", 6);           // 100000000n (USDT)
formatUnits(100000000n, 6);     // "100"
```

### Хеширование

```typescript
import {
  keccak256,
  sha256,
  toHex,
  toBytes,
  encodePacked
} from "viem";

// Keccak256
const hash = keccak256(toHex("Hello"));

// Сигнатура функции
import { getFunctionSelector, getEventSelector } from "viem";

const selector = getFunctionSelector("transfer(address,uint256)");
// "0xa9059cbb"

const eventSig = getEventSelector("Transfer(address,address,uint256)");

// Packed encoding (как abi.encodePacked в Solidity)
const packed = encodePacked(
  ["address", "uint256"],
  ["0x...", 1000n]
);
```

### Работа с адресами

```typescript
import {
  isAddress,
  getAddress,
  isAddressEqual,
  getContractAddress
} from "viem";

// Проверка
isAddress("0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"); // true

// Checksum
getAddress("0x742d35cc6634c0532925a3b844bc9e7595f0ab3d");
// "0x742d35Cc6634C0532925a3b844Bc9e7595f0Ab3d"

// Сравнение (регистронезависимое)
isAddressEqual("0xABC...", "0xabc..."); // true

// Вычисление адреса контракта
const contractAddress = getContractAddress({
  from: "0x...",
  nonce: 5n
});
```

### ABI кодирование

```typescript
import {
  encodeAbiParameters,
  decodeAbiParameters,
  encodeFunctionData,
  decodeFunctionData,
  encodeFunctionResult,
  decodeFunctionResult
} from "viem";

// Кодирование параметров
const encoded = encodeAbiParameters(
  [
    { name: "to", type: "address" },
    { name: "amount", type: "uint256" }
  ],
  ["0x...", 1000n]
);

// Декодирование
const decoded = decodeAbiParameters(
  [
    { name: "to", type: "address" },
    { name: "amount", type: "uint256" }
  ],
  encoded
);

// Кодирование вызова функции
const calldata = encodeFunctionData({
  abi: parseAbi(["function transfer(address to, uint256 amount)"]),
  functionName: "transfer",
  args: ["0x...", 1000n]
});
```

## ENS

```typescript
// Резолв ENS
const address = await publicClient.getEnsAddress({
  name: "vitalik.eth"
});
console.log("Адрес:", address);

// Обратный резолв
const name = await publicClient.getEnsName({
  address: "0x..."
});
console.log("ENS:", name);

// Получение аватара
const avatar = await publicClient.getEnsAvatar({
  name: "vitalik.eth"
});

// Получение текстовой записи
const twitter = await publicClient.getEnsText({
  name: "vitalik.eth",
  key: "com.twitter"
});
```

## Работа с разными сетями

```typescript
import {
  mainnet,
  sepolia,
  polygon,
  arbitrum,
  optimism,
  base
} from "viem/chains";

// Клиент для конкретной сети
const polygonClient = createPublicClient({
  chain: polygon,
  transport: http()
});

// Кастомная сеть
const customChain = {
  id: 12345,
  name: "Custom Network",
  network: "custom",
  nativeCurrency: {
    name: "Custom Token",
    symbol: "CTK",
    decimals: 18
  },
  rpcUrls: {
    default: { http: ["https://rpc.custom.network"] }
  },
  blockExplorers: {
    default: { name: "Explorer", url: "https://explorer.custom.network" }
  }
};

const customClient = createPublicClient({
  chain: customChain,
  transport: http()
});
```

## Обработка ошибок

```typescript
import {
  BaseError,
  ContractFunctionRevertedError,
  InsufficientFundsError,
  UserRejectedRequestError
} from "viem";

try {
  await walletClient.writeContract({
    address: USDT_ADDRESS,
    abi,
    functionName: "transfer",
    args: ["0x...", parseUnits("1000000", 6)]
  });
} catch (error) {
  if (error instanceof ContractFunctionRevertedError) {
    console.error("Контракт отклонил вызов");
    console.error("Причина:", error.reason);
    console.error("Данные:", error.data);
  } else if (error instanceof InsufficientFundsError) {
    console.error("Недостаточно средств");
  } else if (error instanceof UserRejectedRequestError) {
    console.error("Пользователь отклонил транзакцию");
  } else if (error instanceof BaseError) {
    console.error("Viem ошибка:", error.shortMessage);
    console.error("Детали:", error.details);
  } else {
    console.error("Неизвестная ошибка:", error);
  }
}
```

### Декодирование custom errors

```typescript
import { decodeErrorResult } from "viem";

const abi = parseAbi([
  "error InsufficientBalance(uint256 available, uint256 required)",
  "error Unauthorized()"
]);

try {
  await client.simulateContract({...});
} catch (error) {
  if (error.data) {
    const decodedError = decodeErrorResult({
      abi,
      data: error.data
    });

    if (decodedError.errorName === "InsufficientBalance") {
      console.error(
        `Недостаточно: есть ${decodedError.args.available}, нужно ${decodedError.args.required}`
      );
    }
  }
}
```

## Contract Instances

```typescript
import { getContract } from "viem";

// Создание экземпляра контракта
const contract = getContract({
  address: USDT_ADDRESS,
  abi,
  client: {
    public: publicClient,
    wallet: walletClient
  }
});

// Использование
const name = await contract.read.name();
const balance = await contract.read.balanceOf(["0x..."]);

const hash = await contract.write.transfer(["0x...", 1000n]);

// Подписка на события
const unwatch = contract.watchEvent.Transfer({
  onLogs: (logs) => console.log(logs)
});
```

## Сравнение с другими библиотеками

| Характеристика | viem | ethers.js v6 | web3.js v4 |
|---------------|------|--------------|------------|
| Размер (gzip) | ~35 КБ | ~80 КБ | ~150 КБ |
| TypeScript | Отличный | Хороший | Хороший |
| Tree-shaking | Полный | Частичный | Частичный |
| Производительность | Отличная | Хорошая | Средняя |
| Multicall | Встроен | Нет | Нет |
| Ошибки | Понятные | Хорошие | Средние |
| Документация | Отличная | Отличная | Хорошая |

### Преимущества viem

1. **Размер** — самая компактная библиотека
2. **Типобезопасность** — лучшая поддержка TypeScript
3. **Производительность** — оптимизированные операции
4. **Multicall** — встроенная поддержка
5. **Современный API** — функциональный подход
6. **Понятные ошибки** — человекочитаемые сообщения

### Недостатки

1. **Новизна** — меньше примеров в интернете
2. **Отличия от ethers/web3** — требует переучивания
3. **Зависимость от wagmi экосистемы** — хотя работает и отдельно

## Best Practices

### 1. Типизация контрактов

```typescript
// Определение типов из ABI
const abi = [
  {
    name: "transfer",
    type: "function",
    inputs: [
      { name: "to", type: "address" },
      { name: "amount", type: "uint256" }
    ],
    outputs: [{ type: "bool" }]
  }
] as const; // важно: as const

// TypeScript автоматически выведет типы
const result = await client.readContract({
  address: "0x...",
  abi,
  functionName: "transfer", // автодополнение
  args: ["0x...", 100n]    // типизированные аргументы
});
```

### 2. Переиспользование клиентов

```typescript
// Создайте клиенты один раз
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http()
});

// Экспортируйте для использования
export { publicClient };
```

### 3. Обработка ретраев

```typescript
import { createPublicClient, http, fallback } from "viem";

const client = createPublicClient({
  chain: mainnet,
  transport: fallback([
    http("https://mainnet.infura.io/v3/KEY", {
      retryCount: 3,
      retryDelay: 1000
    }),
    http("https://eth-mainnet.alchemyapi.io/v2/KEY"),
    http("https://rpc.ankr.com/eth")
  ])
});
```

### 4. Batch запросы

```typescript
// Используйте multicall для множественных чтений
const results = await client.multicall({
  contracts: addresses.map(addr => ({
    address: TOKEN_ADDRESS,
    abi,
    functionName: "balanceOf",
    args: [addr]
  })),
  allowFailure: true // продолжать при ошибках
});
```

### 5. Симуляция перед записью

```typescript
// Всегда симулируйте перед отправкой
try {
  const { request } = await publicClient.simulateContract({
    address: CONTRACT_ADDRESS,
    abi,
    functionName: "riskyFunction",
    args: [arg1, arg2],
    account
  });

  const hash = await walletClient.writeContract(request);
} catch (error) {
  // Обработка ошибки до траты газа
  console.error("Симуляция не прошла:", error);
}
```

## Интеграция с wagmi

```typescript
// wagmi использует viem под капотом
import { useReadContract, useWriteContract } from "wagmi";

function Component() {
  const { data: balance } = useReadContract({
    address: TOKEN_ADDRESS,
    abi,
    functionName: "balanceOf",
    args: [userAddress]
  });

  const { writeContract } = useWriteContract();

  const handleTransfer = () => {
    writeContract({
      address: TOKEN_ADDRESS,
      abi,
      functionName: "transfer",
      args: [recipient, amount]
    });
  };

  return <button onClick={handleTransfer}>Transfer</button>;
}
```

## Ресурсы

- [Официальная документация](https://viem.sh/)
- [GitHub репозиторий](https://github.com/wevm/viem)
- [Примеры кода](https://viem.sh/docs/introduction.html)
- [wagmi документация](https://wagmi.sh/)
- [Discord сообщество](https://discord.gg/wevm)
- [Миграция с ethers.js](https://viem.sh/docs/ethers-migration.html)
