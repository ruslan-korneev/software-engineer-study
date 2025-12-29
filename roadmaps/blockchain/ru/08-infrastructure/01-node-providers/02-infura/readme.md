# Infura

## Введение

Infura — это один из старейших и наиболее надежных провайдеров инфраструктуры для блокчейна, основанный в 2016 году. С 2019 года входит в состав ConsenSys — крупнейшей блокчейн-компании, создавшей MetaMask. Infura предоставляет доступ к нодам Ethereum и других сетей, обслуживая миллионы запросов ежедневно.

MetaMask, самый популярный криптокошелек с более чем 30 миллионами пользователей, использует Infura в качестве основного провайдера нод, что говорит о высокой надежности сервиса.

## Поддерживаемые сети

### Ethereum
- **Mainnet** — основная сеть Ethereum
- **Sepolia** — тестовая сеть (рекомендуется)
- **Holesky** — тестовая сеть для стейкинга

### Layer 2 решения
- **Arbitrum** — One и Sepolia
- **Optimism** — Mainnet и Sepolia
- **Polygon** — PoS и zkEVM
- **Linea** — L2 от ConsenSys
- **Base** — L2 от Coinbase
- **StarkNet** — ZK rollup

### Другие сети
- **Avalanche** — C-Chain
- **BNB Smart Chain** — Mainnet и Testnet
- **Celo** — Mainnet и Alfajores
- **NEAR** — Mainnet и Testnet
- **Palm** — Mainnet и Testnet

### Специализированные API
- **IPFS** — децентрализованное хранилище
- **Filecoin** — хранение данных

## Основные возможности

### 1. JSON-RPC API

Полная поддержка стандартного Ethereum JSON-RPC:

```javascript
// Основные методы
eth_blockNumber        // Номер последнего блока
eth_getBalance         // Баланс адреса
eth_getTransactionByHash // Информация о транзакции
eth_call               // Вызов read-only функции
eth_sendRawTransaction // Отправка подписанной транзакции
eth_getLogs            // Получение логов/событий
eth_gasPrice           // Текущая цена газа
eth_estimateGas        // Оценка газа для транзакции
```

### 2. WebSocket API

Real-time подписки на события:

```javascript
// Подписки
eth_subscribe("newHeads")           // Новые блоки
eth_subscribe("logs", {filter})     // События контрактов
eth_subscribe("newPendingTransactions") // Pending транзакции
eth_subscribe("syncing")            // Статус синхронизации
```

### 3. IPFS API

Интеграция с децентрализованным хранилищем:

```javascript
// Загрузка файлов
POST /api/v0/add
POST /api/v0/pin/add

// Получение файлов
GET /api/v0/cat
GET /api/v0/get
```

### 4. Gas API

Рекомендации по цене газа:

```javascript
// Получение оптимальной цены газа
GET /v3/YOUR_PROJECT_ID/gas/prices

// Ответ включает:
// - safeLow (медленно, дешево)
// - average (средне)
// - fast (быстро)
// - fastest (очень быстро)
```

## Установка и настройка

### Регистрация и создание проекта

1. Зарегистрируйтесь на [infura.io](https://infura.io)
2. Создайте новый проект (API Key)
3. Выберите сети для использования
4. Сохраните Project ID и API Key Secret

### Базовое подключение

```javascript
// HTTP endpoint
const INFURA_URL = `https://mainnet.infura.io/v3/${PROJECT_ID}`;

// WebSocket endpoint
const INFURA_WS = `wss://mainnet.infura.io/ws/v3/${PROJECT_ID}`;

// С использованием API Key Secret (рекомендуется)
const INFURA_URL_AUTH = `https://${API_KEY}:${API_KEY_SECRET}@mainnet.infura.io/v3/${PROJECT_ID}`;
```

### Конфигурация для разных сетей

```javascript
const networks = {
  // Ethereum
  mainnet: `https://mainnet.infura.io/v3/${PROJECT_ID}`,
  sepolia: `https://sepolia.infura.io/v3/${PROJECT_ID}`,

  // Layer 2
  arbitrum: `https://arbitrum-mainnet.infura.io/v3/${PROJECT_ID}`,
  optimism: `https://optimism-mainnet.infura.io/v3/${PROJECT_ID}`,
  polygon: `https://polygon-mainnet.infura.io/v3/${PROJECT_ID}`,
  linea: `https://linea-mainnet.infura.io/v3/${PROJECT_ID}`,

  // Другие
  avalanche: `https://avalanche-mainnet.infura.io/v3/${PROJECT_ID}`,
  bsc: `https://bsc-mainnet.infura.io/v3/${PROJECT_ID}`,
};
```

## Примеры использования

### Нативный fetch

```javascript
const PROJECT_ID = process.env.INFURA_PROJECT_ID;
const INFURA_URL = `https://mainnet.infura.io/v3/${PROJECT_ID}`;

// Получение номера блока
async function getBlockNumber() {
  const response = await fetch(INFURA_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_blockNumber',
      params: [],
      id: 1,
    }),
  });

  const data = await response.json();
  const blockNumber = parseInt(data.result, 16);
  console.log('Текущий блок:', blockNumber);
  return blockNumber;
}

// Получение баланса
async function getBalance(address) {
  const response = await fetch(INFURA_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_getBalance',
      params: [address, 'latest'],
      id: 1,
    }),
  });

  const data = await response.json();
  const balanceWei = BigInt(data.result);
  const balanceEth = Number(balanceWei) / 1e18;
  console.log('Баланс:', balanceEth, 'ETH');
  return balanceEth;
}

// Получение транзакции
async function getTransaction(txHash) {
  const response = await fetch(INFURA_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_getTransactionByHash',
      params: [txHash],
      id: 1,
    }),
  });

  const data = await response.json();
  console.log('Транзакция:', data.result);
  return data.result;
}
```

### Batch запросы

```javascript
// Отправка нескольких запросов одновременно
async function batchRequest(requests) {
  const response = await fetch(INFURA_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(requests),
  });

  return response.json();
}

// Пример использования
async function getMultipleBalances(addresses) {
  const requests = addresses.map((address, index) => ({
    jsonrpc: '2.0',
    method: 'eth_getBalance',
    params: [address, 'latest'],
    id: index,
  }));

  const results = await batchRequest(requests);

  return results.map((result, index) => ({
    address: addresses[index],
    balance: Number(BigInt(result.result)) / 1e18,
  }));
}
```

### Интеграция с ethers.js

```javascript
import { ethers } from 'ethers';

const PROJECT_ID = process.env.INFURA_PROJECT_ID;

// Использование InfuraProvider (встроенный)
const provider = new ethers.InfuraProvider('mainnet', PROJECT_ID);

// Или через JsonRpcProvider
const jsonRpcProvider = new ethers.JsonRpcProvider(
  `https://mainnet.infura.io/v3/${PROJECT_ID}`
);

// Базовые операции
async function basicOperations() {
  // Номер блока
  const blockNumber = await provider.getBlockNumber();
  console.log('Блок:', blockNumber);

  // Баланс
  const balance = await provider.getBalance('0x...');
  console.log('Баланс:', ethers.formatEther(balance), 'ETH');

  // Информация о блоке
  const block = await provider.getBlock(blockNumber);
  console.log('Блок:', block);

  // Цена газа
  const feeData = await provider.getFeeData();
  console.log('Gas Price:', ethers.formatUnits(feeData.gasPrice, 'gwei'), 'Gwei');
}

// Работа со смарт-контрактами
async function contractInteraction() {
  const contractAddress = '0xdAC17F958D2ee523a2206206994597C13D831ec7'; // USDT
  const abi = [
    'function name() view returns (string)',
    'function symbol() view returns (string)',
    'function decimals() view returns (uint8)',
    'function totalSupply() view returns (uint256)',
    'function balanceOf(address owner) view returns (uint256)',
  ];

  const contract = new ethers.Contract(contractAddress, abi, provider);

  const [name, symbol, decimals, totalSupply] = await Promise.all([
    contract.name(),
    contract.symbol(),
    contract.decimals(),
    contract.totalSupply(),
  ]);

  console.log(`
    Токен: ${name} (${symbol})
    Decimals: ${decimals}
    Total Supply: ${ethers.formatUnits(totalSupply, decimals)}
  `);
}
```

### Интеграция с web3.js

```javascript
import Web3 from 'web3';

const PROJECT_ID = process.env.INFURA_PROJECT_ID;

// HTTP провайдер
const web3Http = new Web3(
  `https://mainnet.infura.io/v3/${PROJECT_ID}`
);

// WebSocket провайдер
const web3Ws = new Web3(
  new Web3.providers.WebsocketProvider(
    `wss://mainnet.infura.io/ws/v3/${PROJECT_ID}`
  )
);

// Базовые операции
async function web3Operations() {
  // Номер блока
  const blockNumber = await web3Http.eth.getBlockNumber();
  console.log('Блок:', blockNumber);

  // Баланс
  const balance = await web3Http.eth.getBalance('0x...');
  console.log('Баланс:', web3Http.utils.fromWei(balance, 'ether'), 'ETH');

  // Транзакция
  const tx = await web3Http.eth.getTransaction('0x...');
  console.log('Транзакция:', tx);
}

// Работа с контрактами
async function web3Contract() {
  const abi = [...]; // ABI контракта
  const address = '0x...';

  const contract = new web3Http.eth.Contract(abi, address);

  // Вызов view функции
  const result = await contract.methods.someFunction().call();
  console.log('Результат:', result);
}
```

### WebSocket подписки

```javascript
import WebSocket from 'ws';

const PROJECT_ID = process.env.INFURA_PROJECT_ID;
const ws = new WebSocket(`wss://mainnet.infura.io/ws/v3/${PROJECT_ID}`);

ws.on('open', () => {
  console.log('Подключено к Infura WebSocket');

  // Подписка на новые блоки
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_subscribe',
    params: ['newHeads'],
    id: 1,
  }));

  // Подписка на логи контракта
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_subscribe',
    params: [
      'logs',
      {
        address: '0xdAC17F958D2ee523a2206206994597C13D831ec7', // USDT
        topics: [
          '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef' // Transfer
        ]
      }
    ],
    id: 2,
  }));
});

ws.on('message', (data) => {
  const message = JSON.parse(data);

  if (message.method === 'eth_subscription') {
    const { subscription, result } = message.params;

    if (result.number) {
      // Новый блок
      console.log('Новый блок:', parseInt(result.number, 16));
    } else if (result.topics) {
      // Событие Transfer
      console.log('Transfer:', {
        from: '0x' + result.topics[1].slice(26),
        to: '0x' + result.topics[2].slice(26),
        transactionHash: result.transactionHash,
      });
    }
  }
});

ws.on('error', (error) => {
  console.error('WebSocket ошибка:', error);
});

ws.on('close', () => {
  console.log('WebSocket закрыт');
});
```

### Работа с IPFS

```javascript
import FormData from 'form-data';
import fs from 'fs';

const PROJECT_ID = process.env.INFURA_IPFS_PROJECT_ID;
const PROJECT_SECRET = process.env.INFURA_IPFS_PROJECT_SECRET;

const auth = Buffer.from(`${PROJECT_ID}:${PROJECT_SECRET}`).toString('base64');

// Загрузка файла в IPFS
async function uploadToIPFS(filePath) {
  const formData = new FormData();
  formData.append('file', fs.createReadStream(filePath));

  const response = await fetch('https://ipfs.infura.io:5001/api/v0/add', {
    method: 'POST',
    headers: {
      'Authorization': `Basic ${auth}`,
    },
    body: formData,
  });

  const data = await response.json();
  console.log('Файл загружен:', data);
  console.log('IPFS Hash:', data.Hash);
  console.log('URL:', `https://ipfs.io/ipfs/${data.Hash}`);

  return data.Hash;
}

// Загрузка JSON в IPFS
async function uploadJSON(jsonData) {
  const formData = new FormData();
  formData.append('file', Buffer.from(JSON.stringify(jsonData)));

  const response = await fetch('https://ipfs.infura.io:5001/api/v0/add', {
    method: 'POST',
    headers: {
      'Authorization': `Basic ${auth}`,
    },
    body: formData,
  });

  const data = await response.json();
  return data.Hash;
}

// Pin файла
async function pinFile(hash) {
  const response = await fetch(
    `https://ipfs.infura.io:5001/api/v0/pin/add?arg=${hash}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${auth}`,
      },
    }
  );

  return response.json();
}

// Получение файла
async function getFromIPFS(hash) {
  const response = await fetch(
    `https://ipfs.infura.io:5001/api/v0/cat?arg=${hash}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${auth}`,
      },
    }
  );

  return response.text();
}

// Пример: загрузка NFT метаданных
async function uploadNFTMetadata(metadata) {
  const ipfsHash = await uploadJSON(metadata);
  return `ipfs://${ipfsHash}`;
}

const nftMetadata = {
  name: 'My NFT',
  description: 'A unique digital asset',
  image: 'ipfs://QmXxx...', // Ссылка на изображение в IPFS
  attributes: [
    { trait_type: 'Color', value: 'Blue' },
    { trait_type: 'Rarity', value: 'Rare' },
  ],
};

const tokenURI = await uploadNFTMetadata(nftMetadata);
console.log('Token URI:', tokenURI);
```

### Отправка транзакций

```javascript
import { ethers } from 'ethers';

const PROJECT_ID = process.env.INFURA_PROJECT_ID;
const PRIVATE_KEY = process.env.PRIVATE_KEY;

const provider = new ethers.InfuraProvider('sepolia', PROJECT_ID);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// Отправка ETH
async function sendETH(toAddress, amountEth) {
  // Получаем текущие параметры газа
  const feeData = await provider.getFeeData();

  const tx = {
    to: toAddress,
    value: ethers.parseEther(amountEth.toString()),
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
  };

  console.log('Отправка транзакции...');
  const txResponse = await wallet.sendTransaction(tx);
  console.log('Hash:', txResponse.hash);

  console.log('Ожидание подтверждения...');
  const receipt = await txResponse.wait();
  console.log('Подтверждена в блоке:', receipt.blockNumber);

  return receipt;
}

// Вызов функции контракта
async function callContractFunction(contractAddress, abi, functionName, args) {
  const contract = new ethers.Contract(contractAddress, abi, wallet);

  // Оценка газа
  const gasEstimate = await contract[functionName].estimateGas(...args);
  console.log('Оценка газа:', gasEstimate.toString());

  // Выполнение транзакции
  const tx = await contract[functionName](...args, {
    gasLimit: gasEstimate * 120n / 100n, // +20% запас
  });

  console.log('Hash:', tx.hash);
  const receipt = await tx.wait();

  return receipt;
}
```

## Аутентификация и безопасность

### API Key Secret

```javascript
// Рекомендуется использовать API Key Secret для аутентификации
const response = await fetch(
  `https://mainnet.infura.io/v3/${PROJECT_ID}`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Basic ${Buffer.from(`${API_KEY}:${API_KEY_SECRET}`).toString('base64')}`,
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_blockNumber',
      params: [],
      id: 1,
    }),
  }
);
```

### Allowlist (белый список)

В настройках проекта можно указать:
- **Contract Addresses** — только указанные контракты
- **User Agents** — только определенные User-Agent
- **Origins** — только определенные домены
- **JWT** — аутентификация через JWT токены

### Переменные окружения

```bash
# .env
INFURA_PROJECT_ID=your_project_id
INFURA_API_KEY_SECRET=your_api_key_secret
INFURA_IPFS_PROJECT_ID=your_ipfs_project_id
INFURA_IPFS_PROJECT_SECRET=your_ipfs_secret
```

```javascript
// config.js
import dotenv from 'dotenv';
dotenv.config();

export const config = {
  projectId: process.env.INFURA_PROJECT_ID,
  apiKeySecret: process.env.INFURA_API_KEY_SECRET,

  getHttpUrl(network = 'mainnet') {
    return `https://${network}.infura.io/v3/${this.projectId}`;
  },

  getWsUrl(network = 'mainnet') {
    return `wss://${network}.infura.io/ws/v3/${this.projectId}`;
  },
};
```

## Тарифные планы

### Free Tier
- **100,000 запросов/день**
- Доступ к основным сетям
- Базовая поддержка через документацию
- 3 проекта

### Developer ($50/месяц)
- **200,000 запросов/день**
- Все сети
- Email поддержка
- 10 проектов
- Расширенная аналитика

### Team ($225/месяц)
- **1,000,000 запросов/день**
- Приоритетная поддержка
- 25 проектов
- SLA гарантии
- Archive данные

### Growth ($1,000/месяц)
- **5,000,000 запросов/день**
- Dedicated поддержка
- Неограниченные проекты
- Custom SLA

### Enterprise (индивидуально)
- Неограниченные запросы
- Выделенная инфраструктура
- Персональный менеджер
- 24/7 поддержка

## Преимущества

1. **Надежность и история**
   - Работает с 2016 года
   - Обслуживает MetaMask
   - Проверенная инфраструктура

2. **Часть экосистемы ConsenSys**
   - Интеграция с MetaMask
   - Поддержка Linea (собственный L2)
   - Доступ к инструментам ConsenSys

3. **IPFS интеграция**
   - Встроенная поддержка IPFS
   - Удобно для NFT проектов
   - Pinning сервис

4. **Широкая поддержка сетей**
   - Ethereum и все основные L2
   - Множество EVM-совместимых сетей
   - Filecoin и NEAR

5. **Безопасность**
   - API Key Secret
   - Allowlist для контрактов и доменов
   - JWT аутентификация

## Недостатки

1. **Ограниченные Enhanced APIs**
   - Нет NFT API как у Alchemy
   - Нет Transfers API
   - Только стандартный JSON-RPC

2. **Webhook отсутствуют**
   - Нет встроенной системы уведомлений
   - Необходимо использовать WebSocket

3. **Лимиты Free Tier**
   - 100K запросов/день — меньше чем у конкурентов
   - Быстро расходуется при активной разработке

4. **Документация**
   - Менее интерактивная чем у Alchemy
   - Меньше примеров кода

## Best Practices

### 1. Использование нескольких провайдеров

```javascript
import { ethers } from 'ethers';

class MultiProvider {
  constructor(providers) {
    this.providers = providers;
    this.currentIndex = 0;
  }

  async call(method, params) {
    for (let i = 0; i < this.providers.length; i++) {
      try {
        const provider = this.providers[this.currentIndex];
        const result = await provider[method](...params);
        return result;
      } catch (error) {
        console.log(`Провайдер ${this.currentIndex} недоступен, переключаемся`);
        this.currentIndex = (this.currentIndex + 1) % this.providers.length;
      }
    }
    throw new Error('Все провайдеры недоступны');
  }
}

const infuraProvider = new ethers.InfuraProvider('mainnet', INFURA_PROJECT_ID);
const alchemyProvider = new ethers.AlchemyProvider('mainnet', ALCHEMY_API_KEY);

const multiProvider = new MultiProvider([infuraProvider, alchemyProvider]);
```

### 2. Rate limiting на клиенте

```javascript
class RateLimiter {
  constructor(maxRequests, windowMs) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = [];
  }

  async acquire() {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      const waitTime = this.windowMs - (now - this.requests[0]);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return this.acquire();
    }

    this.requests.push(now);
    return true;
  }
}

const rateLimiter = new RateLimiter(10, 1000); // 10 запросов в секунду

async function rateLimitedRequest(fn) {
  await rateLimiter.acquire();
  return fn();
}
```

### 3. Кэширование ответов

```javascript
import NodeCache from 'node-cache';

const cache = new NodeCache({
  stdTTL: 12, // 12 секунд (1 блок)
  checkperiod: 12,
});

async function getCachedBlockNumber(provider) {
  const cached = cache.get('blockNumber');
  if (cached) return cached;

  const blockNumber = await provider.getBlockNumber();
  cache.set('blockNumber', blockNumber);
  return blockNumber;
}

// Для неизменяемых данных — дольше TTL
async function getCachedTransaction(provider, txHash) {
  const cached = cache.get(`tx_${txHash}`);
  if (cached) return cached;

  const tx = await provider.getTransaction(txHash);
  if (tx && tx.blockNumber) {
    cache.set(`tx_${txHash}`, tx, 3600); // 1 час для подтвержденных
  }
  return tx;
}
```

### 4. Retry логика

```javascript
async function withRetry(fn, maxRetries = 3, baseDelay = 1000) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      const isRetryable =
        error.code === 'TIMEOUT' ||
        error.code === 'SERVER_ERROR' ||
        error.code === -32005; // rate limit

      if (!isRetryable || i === maxRetries - 1) {
        throw error;
      }

      const delay = baseDelay * Math.pow(2, i);
      console.log(`Попытка ${i + 1} не удалась, ждем ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Использование
const balance = await withRetry(() =>
  provider.getBalance('0x...')
);
```

### 5. Мониторинг использования

```javascript
// Создание wrapper для отслеживания запросов
class MonitoredProvider {
  constructor(provider) {
    this.provider = provider;
    this.requestCount = 0;
    this.errorCount = 0;
    this.startTime = Date.now();
  }

  async call(method, ...args) {
    this.requestCount++;
    try {
      return await this.provider[method](...args);
    } catch (error) {
      this.errorCount++;
      throw error;
    }
  }

  getStats() {
    const elapsed = (Date.now() - this.startTime) / 1000;
    return {
      totalRequests: this.requestCount,
      errors: this.errorCount,
      requestsPerSecond: this.requestCount / elapsed,
      errorRate: this.errorCount / this.requestCount,
    };
  }
}
```

## Полезные ссылки

- [Официальная документация](https://docs.infura.io/)
- [Infura Dashboard](https://infura.io/dashboard)
- [Status Page](https://status.infura.io/)
- [GitHub](https://github.com/INFURA)
- [ConsenSys](https://consensys.net/)
