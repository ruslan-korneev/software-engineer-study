# QuickNode

## Введение

QuickNode — это высокопроизводительная платформа для доступа к блокчейн-инфраструктуре, основанная в 2017 году. Компания специализируется на предоставлении быстрых и надежных нод с акцентом на производительность и низкую задержку. QuickNode поддерживает более 25 блокчейн-сетей и обслуживает тысячи разработчиков и компаний по всему миру.

Ключевое отличие QuickNode — это фокус на скорости: компания утверждает, что их ноды в среднем в 2.5 раза быстрее конкурентов благодаря глобальной сети и оптимизированной инфраструктуре.

## Поддерживаемые сети

QuickNode поддерживает впечатляющий список блокчейнов:

### EVM-совместимые сети
- **Ethereum** — Mainnet, Sepolia, Holesky
- **Polygon** — PoS, zkEVM, Amoy
- **Arbitrum** — One, Nova, Sepolia
- **Optimism** — Mainnet, Sepolia
- **Base** — Mainnet, Sepolia
- **Avalanche** — C-Chain, Fuji
- **BNB Smart Chain** — Mainnet, Testnet
- **Fantom** — Opera, Testnet
- **Gnosis** (xDai)
- **Celo**
- **Harmony**
- **zkSync** — Era, Sepolia
- **Scroll**
- **Linea**
- **Mantle**

### Не-EVM сети
- **Solana** — Mainnet, Devnet, Testnet
- **Bitcoin** — Mainnet, Testnet
- **NEAR** — Mainnet, Testnet
- **Algorand**
- **Stacks**
- **Cosmos** — через IBC
- **Aptos**
- **Sui**
- **Tron**

## Основные возможности

### 1. Core API (JSON-RPC)

Стандартные RPC методы для всех сетей:

```javascript
// Ethereum методы
eth_blockNumber
eth_getBalance
eth_getTransactionByHash
eth_call
eth_sendRawTransaction
eth_getLogs
eth_getBlockByNumber
eth_estimateGas

// Solana методы
getBalance
getAccountInfo
getTransaction
sendTransaction
getSlot
getBlockHeight
```

### 2. QuickNode Add-ons

Расширения для дополнительной функциональности:

- **Token and NFT API** — информация о токенах и NFT
- **Transaction Receipts** — расширенные данные о транзакциях
- **Debug & Trace API** — отладка транзакций
- **Merkle Proofs** — генерация merkle proof
- **Notifications** — webhooks и алерты
- **GraphQL API** — альтернативный способ запросов
- **QuickAlerts** — уведомления о событиях

### 3. Streams

Real-time потоки данных:

```javascript
// Подписка на события в реальном времени
// - Новые блоки
// - Транзакции по адресу
// - События смарт-контрактов
// - Изменения состояния
```

### 4. Functions

Serverless функции для обработки данных:

```javascript
// Пользовательская логика на стороне QuickNode
// Обработка webhook событий
// Трансформация данных
// Агрегация и фильтрация
```

## Установка и настройка

### Создание эндпоинта

1. Зарегистрируйтесь на [quicknode.com](https://www.quicknode.com/)
2. Нажмите "Create Endpoint"
3. Выберите сеть и регион
4. Настройте Add-ons (опционально)
5. Скопируйте HTTP и WSS URLs

### Формат эндпоинтов

```javascript
// HTTP endpoint
const QUICKNODE_HTTP = 'https://your-endpoint-name.quiknode.pro/your-token/';

// WebSocket endpoint
const QUICKNODE_WS = 'wss://your-endpoint-name.quiknode.pro/your-token/';

// С аутентификацией (опционально)
const QUICKNODE_AUTH = 'https://your-endpoint-name.quiknode.pro/your-token/?auth=your-auth-token';
```

### Базовая конфигурация

```javascript
import { ethers } from 'ethers';

// Конфигурация
const config = {
  ethereum: {
    http: process.env.QUICKNODE_ETH_HTTP,
    ws: process.env.QUICKNODE_ETH_WS,
  },
  polygon: {
    http: process.env.QUICKNODE_POLYGON_HTTP,
    ws: process.env.QUICKNODE_POLYGON_WS,
  },
  solana: {
    http: process.env.QUICKNODE_SOLANA_HTTP,
    ws: process.env.QUICKNODE_SOLANA_WS,
  },
};

// Создание провайдеров
const ethProvider = new ethers.JsonRpcProvider(config.ethereum.http);
const polygonProvider = new ethers.JsonRpcProvider(config.polygon.http);
```

## Примеры использования

### Базовые операции с Ethereum

```javascript
import { ethers } from 'ethers';

const QUICKNODE_URL = process.env.QUICKNODE_ETH_HTTP;
const provider = new ethers.JsonRpcProvider(QUICKNODE_URL);

// Получение информации о блоке
async function getBlockInfo() {
  const blockNumber = await provider.getBlockNumber();
  console.log('Текущий блок:', blockNumber);

  const block = await provider.getBlock(blockNumber);
  console.log('Информация о блоке:', {
    hash: block.hash,
    timestamp: new Date(block.timestamp * 1000),
    transactions: block.transactions.length,
    gasUsed: block.gasUsed.toString(),
    gasLimit: block.gasLimit.toString(),
  });

  return block;
}

// Получение баланса
async function getBalance(address) {
  const balance = await provider.getBalance(address);
  console.log('Баланс:', ethers.formatEther(balance), 'ETH');
  return balance;
}

// Получение nonce
async function getNonce(address) {
  const nonce = await provider.getTransactionCount(address);
  console.log('Nonce:', nonce);
  return nonce;
}

// Получение кода контракта
async function getCode(address) {
  const code = await provider.getCode(address);
  const isContract = code !== '0x';
  console.log('Это контракт:', isContract);
  return code;
}
```

### Работа с транзакциями

```javascript
// Получение транзакции
async function getTransaction(txHash) {
  const tx = await provider.getTransaction(txHash);

  if (tx) {
    console.log('Транзакция:', {
      hash: tx.hash,
      from: tx.from,
      to: tx.to,
      value: ethers.formatEther(tx.value),
      gasPrice: ethers.formatUnits(tx.gasPrice, 'gwei'),
      nonce: tx.nonce,
      blockNumber: tx.blockNumber,
    });
  }

  return tx;
}

// Получение чека транзакции
async function getReceipt(txHash) {
  const receipt = await provider.getTransactionReceipt(txHash);

  if (receipt) {
    console.log('Чек транзакции:', {
      status: receipt.status === 1 ? 'Успешно' : 'Неудачно',
      blockNumber: receipt.blockNumber,
      gasUsed: receipt.gasUsed.toString(),
      effectiveGasPrice: ethers.formatUnits(receipt.gasPrice, 'gwei'),
      logs: receipt.logs.length,
    });
  }

  return receipt;
}

// Отправка транзакции
async function sendTransaction(wallet, to, value) {
  const tx = await wallet.sendTransaction({
    to,
    value: ethers.parseEther(value.toString()),
  });

  console.log('Отправлено:', tx.hash);

  const receipt = await tx.wait();
  console.log('Подтверждено в блоке:', receipt.blockNumber);

  return receipt;
}
```

### Token and NFT API (Add-on)

```javascript
// Получение балансов токенов
async function getTokenBalances(address) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'qn_getWalletTokenBalance',
      params: [{
        wallet: address,
        perPage: 100,
      }],
      id: 1,
    }),
  });

  const data = await response.json();

  console.log('Токены:', data.result.assets);
  return data.result;
}

// Получение NFT кошелька
async function getWalletNFTs(address) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'qn_fetchNFTs',
      params: [{
        wallet: address,
        omitFields: ['traits'],
        perPage: 100,
      }],
      id: 1,
    }),
  });

  const data = await response.json();

  for (const nft of data.result.assets) {
    console.log(`
      Коллекция: ${nft.collectionName}
      Название: ${nft.name}
      Token ID: ${nft.tokenId}
      Контракт: ${nft.collectionAddress}
    `);
  }

  return data.result;
}

// Получение метаданных NFT
async function getNFTMetadata(contractAddress, tokenId) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'qn_fetchNFTsByCollection',
      params: [{
        collection: contractAddress,
        tokens: [tokenId],
      }],
      id: 1,
    }),
  });

  const data = await response.json();
  return data.result;
}

// История трансферов NFT
async function getNFTTransfers(contractAddress, tokenId) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'qn_getTransfersByNFT',
      params: [{
        collection: contractAddress,
        tokenId,
        perPage: 50,
      }],
      id: 1,
    }),
  });

  const data = await response.json();
  return data.result;
}
```

### Trace и Debug API (Add-on)

```javascript
// Трассировка транзакции
async function traceTransaction(txHash) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'trace_transaction',
      params: [txHash],
      id: 1,
    }),
  });

  const data = await response.json();
  return data.result;
}

// Debug трассировка с параметрами
async function debugTraceTransaction(txHash) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      jsonrpc: '2.0',
      method: 'debug_traceTransaction',
      params: [
        txHash,
        {
          tracer: 'callTracer',
          tracerConfig: {
            onlyTopCall: false,
            withLog: true,
          },
        },
      ],
      id: 1,
    }),
  });

  const data = await response.json();
  return data.result;
}

// Получение внутренних транзакций
async function getInternalTransactions(txHash) {
  const trace = await traceTransaction(txHash);

  const internalTxs = trace.filter(t => t.type === 'call');

  for (const tx of internalTxs) {
    console.log(`
      От: ${tx.action.from}
      Кому: ${tx.action.to}
      Значение: ${ethers.formatEther(BigInt(tx.action.value || 0))} ETH
      Тип: ${tx.action.callType}
    `);
  }

  return internalTxs;
}
```

### WebSocket подписки

```javascript
import WebSocket from 'ws';

const QUICKNODE_WS = process.env.QUICKNODE_ETH_WS;

function createWebSocketConnection() {
  const ws = new WebSocket(QUICKNODE_WS);

  ws.on('open', () => {
    console.log('WebSocket подключен');

    // Подписка на новые блоки
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_subscribe',
      params: ['newHeads'],
      id: 1,
    }));

    // Подписка на pending транзакции (если включено)
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      method: 'eth_subscribe',
      params: ['newPendingTransactions'],
      id: 2,
    }));
  });

  ws.on('message', (data) => {
    const message = JSON.parse(data);

    if (message.method === 'eth_subscription') {
      handleSubscription(message.params);
    } else if (message.result) {
      console.log('Подписка создана:', message.result);
    }
  });

  ws.on('error', (error) => {
    console.error('WebSocket ошибка:', error);
  });

  ws.on('close', () => {
    console.log('WebSocket закрыт, переподключение через 5 сек...');
    setTimeout(createWebSocketConnection, 5000);
  });

  return ws;
}

function handleSubscription(params) {
  const { subscription, result } = params;

  if (result.number) {
    // Новый блок
    console.log('Новый блок:', parseInt(result.number, 16), {
      hash: result.hash,
      miner: result.miner,
      gasUsed: parseInt(result.gasUsed, 16),
    });
  } else if (typeof result === 'string' && result.startsWith('0x')) {
    // Pending транзакция
    console.log('Pending TX:', result);
  }
}

// Подписка на события контракта
function subscribeToContractEvents(ws, contractAddress, topic) {
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_subscribe',
    params: [
      'logs',
      {
        address: contractAddress,
        topics: [topic],
      },
    ],
    id: 3,
  }));
}
```

### Работа с Solana

```javascript
import { Connection, PublicKey, LAMPORTS_PER_SOL } from '@solana/web3.js';

const QUICKNODE_SOLANA = process.env.QUICKNODE_SOLANA_HTTP;
const connection = new Connection(QUICKNODE_SOLANA, 'confirmed');

// Получение баланса SOL
async function getSolBalance(address) {
  const publicKey = new PublicKey(address);
  const balance = await connection.getBalance(publicKey);
  console.log('Баланс:', balance / LAMPORTS_PER_SOL, 'SOL');
  return balance;
}

// Получение информации об аккаунте
async function getAccountInfo(address) {
  const publicKey = new PublicKey(address);
  const info = await connection.getAccountInfo(publicKey);

  console.log('Информация об аккаунте:', {
    lamports: info?.lamports,
    owner: info?.owner.toString(),
    executable: info?.executable,
    rentEpoch: info?.rentEpoch,
  });

  return info;
}

// Получение последних транзакций
async function getRecentTransactions(address, limit = 10) {
  const publicKey = new PublicKey(address);
  const signatures = await connection.getSignaturesForAddress(publicKey, { limit });

  for (const sig of signatures) {
    console.log(`
      Подпись: ${sig.signature}
      Слот: ${sig.slot}
      Время: ${new Date(sig.blockTime * 1000)}
      Ошибка: ${sig.err ? 'Да' : 'Нет'}
    `);
  }

  return signatures;
}

// Получение информации о транзакции
async function getTransaction(signature) {
  const tx = await connection.getTransaction(signature, {
    maxSupportedTransactionVersion: 0,
  });

  if (tx) {
    console.log('Транзакция:', {
      slot: tx.slot,
      fee: tx.meta?.fee,
      status: tx.meta?.err ? 'Ошибка' : 'Успешно',
    });
  }

  return tx;
}

// Получение токенов SPL
async function getSPLTokens(address) {
  const publicKey = new PublicKey(address);
  const tokens = await connection.getParsedTokenAccountsByOwner(publicKey, {
    programId: new PublicKey('TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA'),
  });

  for (const token of tokens.value) {
    const info = token.account.data.parsed.info;
    console.log(`
      Mint: ${info.mint}
      Баланс: ${info.tokenAmount.uiAmount}
      Decimals: ${info.tokenAmount.decimals}
    `);
  }

  return tokens;
}
```

### QuickAlerts (Webhooks)

```javascript
// Настройка webhook на стороне QuickNode Dashboard
// или через API:

async function createQuickAlert() {
  const response = await fetch('https://api.quicknode.com/quickalerts/rest/v1/alerts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.QUICKNODE_API_KEY,
    },
    body: JSON.stringify({
      name: 'My Alert',
      expression: 'tx_to == "0x..."',
      network: 'ethereum-mainnet',
      destinationIds: ['webhook-destination-id'],
    }),
  });

  return response.json();
}

// Обработка webhook
import express from 'express';

const app = express();
app.use(express.json());

app.post('/webhook/quicknode', (req, res) => {
  const events = req.body;

  for (const event of events) {
    console.log('Событие:', {
      blockNumber: event.blockNumber,
      transactionHash: event.transactionHash,
      from: event.from,
      to: event.to,
      value: event.value,
    });

    // Обработка события
    processEvent(event);
  }

  res.status(200).send('OK');
});

function processEvent(event) {
  // Ваша логика обработки
}

app.listen(3000);
```

### Streams API

```javascript
// Создание stream через API
async function createStream() {
  const response = await fetch('https://api.quicknode.com/streams/rest/v1/streams', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.QUICKNODE_API_KEY,
    },
    body: JSON.stringify({
      name: 'My Stream',
      network: 'ethereum-mainnet',
      dataset: 'block_with_receipts',
      filter_function: `
        function main(data) {
          const block = data.block;
          const transactions = block.transactions.filter(tx =>
            tx.to && tx.to.toLowerCase() === '0x...'.toLowerCase()
          );
          return {
            blockNumber: block.number,
            transactions: transactions.length,
            data: transactions
          };
        }
      `,
      destination: {
        type: 'webhook',
        url: 'https://your-server.com/webhook',
      },
    }),
  });

  return response.json();
}

// Получение данных из stream
app.post('/webhook/stream', (req, res) => {
  const { blockNumber, transactions, data } = req.body;

  console.log(`Блок ${blockNumber}: ${transactions} транзакций`);

  for (const tx of data) {
    console.log('TX:', tx.hash);
  }

  res.status(200).send('OK');
});
```

## Тарифные планы

### Free Tier
- **10M credits/месяц**
- 1 эндпоинт
- Базовые RPC методы
- Community поддержка

### Starter ($49/месяц)
- **200M credits/месяц**
- 3 эндпоинта
- Add-ons включены
- Email поддержка
- Websocket

### Growth ($299/месяц)
- **3B credits/месяц**
- 10 эндпоинтов
- Приоритетная поддержка
- Archive данные
- SLA 99.9%

### Business ($599+/месяц)
- **5B+ credits/месяц**
- Неограниченные эндпоинты
- Dedicated поддержка
- Custom SLA

### Enterprise (индивидуально)
- Неограниченные credits
- Выделенная инфраструктура
- Персональный менеджер
- 24/7 поддержка

## Преимущества

1. **Высокая скорость**
   - Глобальная сеть серверов
   - Оптимизированная маршрутизация
   - Низкая задержка

2. **Множество блокчейнов**
   - 25+ поддерживаемых сетей
   - EVM и не-EVM
   - Быстрое добавление новых

3. **Add-ons экосистема**
   - NFT и Token API
   - Trace и Debug
   - GraphQL
   - Streams

4. **Удобный Dashboard**
   - Аналитика в реальном времени
   - Мониторинг использования
   - Простое управление

5. **Гибкость**
   - Выбор региона для эндпоинта
   - Кастомизация Add-ons
   - API для управления

## Недостатки

1. **Credit система**
   - Сложнее прогнозировать расходы
   - Разные операции = разная стоимость
   - Add-ons увеличивают потребление

2. **Add-ons за дополнительную плату**
   - Многие функции требуют Add-ons
   - Увеличивает итоговую стоимость

3. **Документация**
   - Иногда устаревшая
   - Меньше примеров чем у Alchemy

4. **WebSocket лимиты**
   - Ограничения на количество подписок
   - Может быть нестабильным

## Best Practices

### 1. Оптимизация использования credits

```javascript
// Разные методы имеют разную стоимость в credits
// eth_blockNumber - 1 credit
// eth_getBalance - 1 credit
// eth_call - 5 credits
// trace_transaction - 100+ credits

// Плохо: много трассировок
for (const tx of transactions) {
  await traceTransaction(tx.hash); // 100+ credits каждая!
}

// Хорошо: фильтрация перед трассировкой
const suspiciousTx = transactions.filter(tx =>
  tx.value > ethers.parseEther('100')
);

for (const tx of suspiciousTx) {
  await traceTransaction(tx.hash);
}
```

### 2. Выбор правильного региона

```javascript
// Выбирайте регион ближе к вашим пользователям
// US-East - для американских пользователей
// EU-West - для европейских
// Asia-Pacific - для азиатских

// Создавайте несколько эндпоинтов для geo-routing
const endpoints = {
  us: process.env.QUICKNODE_US,
  eu: process.env.QUICKNODE_EU,
  asia: process.env.QUICKNODE_ASIA,
};

function getEndpoint(userRegion) {
  return endpoints[userRegion] || endpoints.us;
}
```

### 3. Кэширование и батчинг

```javascript
import NodeCache from 'node-cache';

const cache = new NodeCache({ stdTTL: 12 });

// Кэширование
async function getCached(key, fetcher, ttl) {
  let data = cache.get(key);
  if (data) return data;

  data = await fetcher();
  cache.set(key, data, ttl);
  return data;
}

// Batch запросы
async function batchRPC(requests) {
  const response = await fetch(QUICKNODE_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(
      requests.map((req, i) => ({
        jsonrpc: '2.0',
        method: req.method,
        params: req.params,
        id: i,
      }))
    ),
  });

  return response.json();
}

// Использование batch
const results = await batchRPC([
  { method: 'eth_blockNumber', params: [] },
  { method: 'eth_gasPrice', params: [] },
  { method: 'eth_getBalance', params: ['0x...', 'latest'] },
]);
```

### 4. Мониторинг через Dashboard API

```javascript
async function getUsageStats() {
  const response = await fetch(
    'https://api.quicknode.com/analytics/rest/v1/usage',
    {
      headers: {
        'x-api-key': process.env.QUICKNODE_API_KEY,
      },
    }
  );

  const stats = await response.json();

  console.log('Использование credits:', {
    used: stats.creditsUsed,
    limit: stats.creditsLimit,
    remaining: stats.creditsLimit - stats.creditsUsed,
    percentUsed: (stats.creditsUsed / stats.creditsLimit * 100).toFixed(2) + '%',
  });

  return stats;
}

// Настройка алертов при достижении лимита
async function checkAndAlert() {
  const stats = await getUsageStats();
  const percentUsed = stats.creditsUsed / stats.creditsLimit * 100;

  if (percentUsed > 80) {
    console.warn('ВНИМАНИЕ: Использовано более 80% credits!');
    // Отправка уведомления
  }
}
```

### 5. Обработка ошибок и retry

```javascript
class QuickNodeClient {
  constructor(url, options = {}) {
    this.url = url;
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;
  }

  async request(method, params) {
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        const response = await fetch(this.url, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            jsonrpc: '2.0',
            method,
            params,
            id: Date.now(),
          }),
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        const data = await response.json();

        if (data.error) {
          throw new Error(data.error.message);
        }

        return data.result;
      } catch (error) {
        const isLastAttempt = attempt === this.maxRetries - 1;

        if (isLastAttempt) {
          throw error;
        }

        console.log(`Попытка ${attempt + 1} не удалась: ${error.message}`);
        await new Promise(r => setTimeout(r, this.retryDelay * (attempt + 1)));
      }
    }
  }
}

const client = new QuickNodeClient(QUICKNODE_URL, {
  maxRetries: 3,
  retryDelay: 1000,
});

const balance = await client.request('eth_getBalance', ['0x...', 'latest']);
```

## Полезные ссылки

- [Официальная документация](https://www.quicknode.com/docs)
- [QuickNode Dashboard](https://dashboard.quicknode.com/)
- [Guides](https://www.quicknode.com/guides)
- [Status Page](https://status.quicknode.com/)
- [GitHub](https://github.com/quiknode-labs)
- [Discord](https://discord.gg/quicknode)
