# Alchemy

## Введение

Alchemy — это ведущая платформа для разработки Web3-приложений, которая предоставляет инфраструктуру для взаимодействия с блокчейн-сетями. Компания была основана в 2017 году и быстро стала одним из самых популярных провайдеров нод в индустрии, обслуживая такие проекты как OpenSea, Aave, Dapper Labs и многие другие.

## Поддерживаемые сети

Alchemy поддерживает широкий спектр блокчейн-сетей:

### Основные сети (Mainnet)
- **Ethereum** — полная поддержка всех RPC-методов
- **Polygon** — включая Polygon PoS и zkEVM
- **Arbitrum** — One и Nova
- **Optimism** — Layer 2 решение
- **Base** — L2 от Coinbase
- **Solana** — высокопроизводительная сеть
- **Astar** — мультичейн dApp хаб

### Тестовые сети (Testnet)
- Sepolia (Ethereum)
- Goerli (deprecated)
- Mumbai (Polygon)
- Arbitrum Goerli
- Optimism Goerli

## Основные возможности

### 1. Supernode

Supernode — это запатентованная технология Alchemy, которая обеспечивает:
- **Высокую доступность** — 99.9% uptime
- **Автоматическое масштабирование** — обработка пиковых нагрузок
- **Низкую задержку** — оптимизированная маршрутизация запросов
- **Консистентность данных** — синхронизация между нодами

### 2. Enhanced APIs

Расширенные API, которые упрощают разработку:

```javascript
// Получение всех NFT кошелька (одним запросом!)
const nfts = await alchemy.nft.getNftsForOwner("0x...");

// Получение балансов всех токенов
const balances = await alchemy.core.getTokenBalances("0x...");

// История транзакций
const transfers = await alchemy.core.getAssetTransfers({
  fromAddress: "0x...",
  category: ["external", "erc20", "erc721"]
});
```

### 3. Alchemy Notify (Webhooks)

Система уведомлений в реальном времени:
- **Mined Transactions** — уведомления о подтвержденных транзакциях
- **Dropped Transactions** — отслеживание отклоненных транзакций
- **Address Activity** — активность по адресу
- **NFT Activity** — события с NFT
- **Custom Webhooks** — настраиваемые условия

### 4. Alchemy Monitor

Панель мониторинга включает:
- Аналитику использования API
- Отслеживание ошибок
- Метрики производительности
- Алерты и уведомления

## Установка и настройка

### Установка SDK

```bash
# npm
npm install alchemy-sdk

# yarn
yarn add alchemy-sdk

# pnpm
pnpm add alchemy-sdk
```

### Базовая конфигурация

```javascript
import { Alchemy, Network } from 'alchemy-sdk';

const config = {
  apiKey: process.env.ALCHEMY_API_KEY,
  network: Network.ETH_MAINNET,
};

const alchemy = new Alchemy(config);
```

### Конфигурация для разных сетей

```javascript
// Ethereum Mainnet
const ethMainnet = new Alchemy({
  apiKey: process.env.ALCHEMY_API_KEY,
  network: Network.ETH_MAINNET,
});

// Polygon Mainnet
const polygon = new Alchemy({
  apiKey: process.env.ALCHEMY_API_KEY,
  network: Network.MATIC_MAINNET,
});

// Arbitrum
const arbitrum = new Alchemy({
  apiKey: process.env.ALCHEMY_API_KEY,
  network: Network.ARB_MAINNET,
});

// Optimism
const optimism = new Alchemy({
  apiKey: process.env.ALCHEMY_API_KEY,
  network: Network.OPT_MAINNET,
});
```

## Примеры использования

### Базовые операции

```javascript
import { Alchemy, Network, Utils } from 'alchemy-sdk';

const alchemy = new Alchemy({
  apiKey: 'YOUR_API_KEY',
  network: Network.ETH_MAINNET,
});

// Получение текущего номера блока
async function getBlockNumber() {
  const blockNumber = await alchemy.core.getBlockNumber();
  console.log('Текущий блок:', blockNumber);
  return blockNumber;
}

// Получение баланса адреса
async function getBalance(address) {
  const balance = await alchemy.core.getBalance(address);
  console.log('Баланс:', Utils.formatEther(balance), 'ETH');
  return balance;
}

// Получение информации о транзакции
async function getTransaction(txHash) {
  const tx = await alchemy.core.getTransaction(txHash);
  console.log('Транзакция:', tx);
  return tx;
}

// Получение чека транзакции
async function getTransactionReceipt(txHash) {
  const receipt = await alchemy.core.getTransactionReceipt(txHash);
  console.log('Чек транзакции:', receipt);
  return receipt;
}
```

### Работа с NFT

```javascript
// Получение всех NFT кошелька
async function getWalletNFTs(address) {
  const nfts = await alchemy.nft.getNftsForOwner(address);

  console.log(`Найдено ${nfts.totalCount} NFT:`);

  for (const nft of nfts.ownedNfts) {
    console.log(`
      Название: ${nft.title}
      Контракт: ${nft.contract.address}
      Token ID: ${nft.tokenId}
      Описание: ${nft.description}
    `);
  }

  return nfts;
}

// Получение метаданных конкретного NFT
async function getNFTMetadata(contractAddress, tokenId) {
  const metadata = await alchemy.nft.getNftMetadata(
    contractAddress,
    tokenId
  );

  console.log('Метаданные NFT:', metadata);
  return metadata;
}

// Получение владельцев NFT коллекции
async function getCollectionOwners(contractAddress) {
  const owners = await alchemy.nft.getOwnersForContract(contractAddress);
  console.log(`Уникальных владельцев: ${owners.owners.length}`);
  return owners;
}

// Получение floor price коллекции
async function getFloorPrice(contractAddress) {
  const floorPrice = await alchemy.nft.getFloorPrice(contractAddress);
  console.log('Floor price:', floorPrice);
  return floorPrice;
}
```

### Работа с токенами (ERC-20)

```javascript
// Получение балансов всех токенов
async function getAllTokenBalances(address) {
  const balances = await alchemy.core.getTokenBalances(address);

  // Фильтруем нулевые балансы
  const nonZeroBalances = balances.tokenBalances.filter(
    token => token.tokenBalance !== '0x0000000000000000000000000000000000000000000000000000000000000000'
  );

  console.log(`Найдено ${nonZeroBalances.length} токенов с балансом`);

  // Получаем метаданные для каждого токена
  for (const token of nonZeroBalances) {
    const metadata = await alchemy.core.getTokenMetadata(token.contractAddress);

    const balance = parseInt(token.tokenBalance, 16) / Math.pow(10, metadata.decimals);

    console.log(`${metadata.symbol}: ${balance}`);
  }

  return nonZeroBalances;
}

// Получение метаданных токена
async function getTokenMetadata(contractAddress) {
  const metadata = await alchemy.core.getTokenMetadata(contractAddress);
  console.log('Токен:', metadata.name);
  console.log('Символ:', metadata.symbol);
  console.log('Decimals:', metadata.decimals);
  return metadata;
}
```

### Asset Transfers API

```javascript
// Получение истории трансферов
async function getTransferHistory(address) {
  const transfers = await alchemy.core.getAssetTransfers({
    fromBlock: '0x0',
    toBlock: 'latest',
    fromAddress: address,
    category: ['external', 'erc20', 'erc721', 'erc1155'],
    withMetadata: true,
    maxCount: 100,
  });

  console.log(`Найдено ${transfers.transfers.length} трансферов`);

  for (const transfer of transfers.transfers) {
    console.log(`
      Хеш: ${transfer.hash}
      От: ${transfer.from}
      Кому: ${transfer.to}
      Актив: ${transfer.asset}
      Значение: ${transfer.value}
      Категория: ${transfer.category}
      Блок: ${transfer.blockNum}
    `);
  }

  return transfers;
}

// Получение входящих трансферов
async function getIncomingTransfers(address) {
  const transfers = await alchemy.core.getAssetTransfers({
    fromBlock: '0x0',
    toBlock: 'latest',
    toAddress: address,
    category: ['external', 'erc20'],
    withMetadata: true,
  });

  return transfers;
}
```

### WebSocket подписки

```javascript
// Подписка на новые блоки
function subscribeToBlocks() {
  alchemy.ws.on('block', (blockNumber) => {
    console.log('Новый блок:', blockNumber);
  });
}

// Подписка на pending транзакции
function subscribeToPendingTransactions() {
  alchemy.ws.on(
    {
      method: 'alchemy_pendingTransactions',
      fromAddress: '0x...', // опционально
      toAddress: '0x...',   // опционально
    },
    (tx) => {
      console.log('Pending транзакция:', tx.hash);
    }
  );
}

// Подписка на события контракта
function subscribeToContractEvents(contractAddress) {
  const filter = {
    address: contractAddress,
    topics: [
      Utils.id('Transfer(address,address,uint256)')
    ]
  };

  alchemy.ws.on(filter, (log) => {
    console.log('Событие Transfer:', log);
  });
}

// Отписка
function unsubscribe() {
  alchemy.ws.removeAllListeners();
}
```

### Интеграция с ethers.js

```javascript
import { ethers } from 'ethers';
import { Alchemy, Network } from 'alchemy-sdk';

// Создание провайдера через URL
const provider = new ethers.JsonRpcProvider(
  `https://eth-mainnet.g.alchemy.com/v2/${process.env.ALCHEMY_API_KEY}`
);

// Или использование AlchemyProvider
const alchemyProvider = new ethers.AlchemyProvider(
  'mainnet',
  process.env.ALCHEMY_API_KEY
);

// Создание кошелька
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

// Отправка транзакции
async function sendTransaction() {
  const tx = await wallet.sendTransaction({
    to: '0x...',
    value: ethers.parseEther('0.1'),
  });

  console.log('Хеш транзакции:', tx.hash);

  const receipt = await tx.wait();
  console.log('Подтверждена в блоке:', receipt.blockNumber);

  return receipt;
}

// Работа со смарт-контрактами
async function interactWithContract() {
  const contractAddress = '0x...';
  const abi = [...]; // ABI контракта

  const contract = new ethers.Contract(contractAddress, abi, wallet);

  // Чтение
  const value = await contract.someReadFunction();

  // Запись
  const tx = await contract.someWriteFunction(arg1, arg2);
  await tx.wait();
}
```

## Настройка Webhooks

### Создание Webhook через Dashboard

1. Перейдите в Alchemy Dashboard
2. Выберите раздел "Notify"
3. Нажмите "Create Webhook"
4. Выберите тип уведомления
5. Укажите URL для получения уведомлений
6. Настройте фильтры

### Обработка Webhook на сервере

```javascript
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

// Верификация подписи Alchemy
function isValidSignature(body, signature, signingKey) {
  const hmac = crypto.createHmac('sha256', signingKey);
  hmac.update(JSON.stringify(body));
  const digest = hmac.digest('hex');
  return signature === digest;
}

// Обработчик webhook
app.post('/webhook/alchemy', (req, res) => {
  const signature = req.headers['x-alchemy-signature'];

  if (!isValidSignature(req.body, signature, process.env.ALCHEMY_SIGNING_KEY)) {
    return res.status(401).send('Invalid signature');
  }

  const { webhookId, id, createdAt, type, event } = req.body;

  console.log('Получен webhook:', {
    type,
    event,
    createdAt,
  });

  // Обработка разных типов событий
  switch (type) {
    case 'MINED_TRANSACTION':
      handleMinedTransaction(event);
      break;
    case 'DROPPED_TRANSACTION':
      handleDroppedTransaction(event);
      break;
    case 'ADDRESS_ACTIVITY':
      handleAddressActivity(event);
      break;
    case 'NFT_ACTIVITY':
      handleNFTActivity(event);
      break;
    default:
      console.log('Неизвестный тип события');
  }

  res.status(200).send('OK');
});

function handleMinedTransaction(event) {
  console.log('Транзакция подтверждена:', event.transaction.hash);
}

function handleAddressActivity(event) {
  for (const activity of event.activity) {
    console.log(`
      Актив: ${activity.asset}
      От: ${activity.fromAddress}
      Кому: ${activity.toAddress}
      Значение: ${activity.value}
    `);
  }
}

app.listen(3000, () => {
  console.log('Webhook сервер запущен на порту 3000');
});
```

## Тарифные планы

### Free Tier
- **300M compute units/месяц**
- Доступ ко всем Enhanced APIs
- 5 приложений
- Базовая поддержка

### Growth ($49/месяц)
- **400M compute units/месяц**
- Приоритетная поддержка
- Расширенная аналитика
- 15 приложений

### Scale ($199/месяц)
- **1.5B compute units/месяц**
- SLA гарантии
- Dedicated поддержка
- Неограниченные приложения

### Enterprise (индивидуально)
- Неограниченные compute units
- Выделенная инфраструктура
- Персональный менеджер
- Custom SLA

## Преимущества

1. **Мощные Enhanced APIs**
   - NFT API — получение всех NFT одним запросом
   - Token API — балансы и метаданные токенов
   - Transfers API — полная история транзакций

2. **Высокая надежность**
   - 99.9% uptime SLA
   - Автоматическое failover
   - Глобальная инфраструктура

3. **Отличная документация**
   - Подробные примеры
   - Интерактивные туториалы
   - SDK для разных языков

4. **Developer Experience**
   - Удобный Dashboard
   - Детальная аналитика
   - Система алертов

5. **Webhooks (Notify)**
   - Real-time уведомления
   - Гибкая настройка
   - Надежная доставка

## Недостатки

1. **Стоимость при масштабировании**
   - Compute units расходуются быстро
   - Enhanced APIs дороже стандартных RPC

2. **Vendor lock-in**
   - Привязка к проприетарным API
   - Сложнее мигрировать на другого провайдера

3. **Лимиты Free Tier**
   - Ограниченное количество запросов
   - Некоторые функции недоступны

4. **Региональные ограничения**
   - Оптимизирован для US/EU
   - Может быть задержка в других регионах

## Best Practices

### 1. Оптимизация использования Compute Units

```javascript
// Плохо: много отдельных запросов
for (const address of addresses) {
  const balance = await alchemy.core.getBalance(address);
}

// Хорошо: batch запросы
const balances = await Promise.all(
  addresses.map(addr => alchemy.core.getBalance(addr))
);
```

### 2. Кэширование результатов

```javascript
import NodeCache from 'node-cache';

const cache = new NodeCache({ stdTTL: 60 }); // TTL 60 секунд

async function getCachedBalance(address) {
  const cacheKey = `balance_${address}`;

  let balance = cache.get(cacheKey);

  if (balance === undefined) {
    balance = await alchemy.core.getBalance(address);
    cache.set(cacheKey, balance);
  }

  return balance;
}
```

### 3. Обработка ошибок

```javascript
async function safeApiCall(apiFunction, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await apiFunction();
    } catch (error) {
      if (error.code === 429) {
        // Rate limit - ждем и пробуем снова
        await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
        continue;
      }

      if (i === retries - 1) throw error;
    }
  }
}

// Использование
const balance = await safeApiCall(() =>
  alchemy.core.getBalance('0x...')
);
```

### 4. Использование переменных окружения

```javascript
// .env
ALCHEMY_API_KEY=your_api_key_here
ALCHEMY_NETWORK=mainnet

// config.js
import { Alchemy, Network } from 'alchemy-sdk';
import dotenv from 'dotenv';

dotenv.config();

const networkMap = {
  mainnet: Network.ETH_MAINNET,
  sepolia: Network.ETH_SEPOLIA,
  polygon: Network.MATIC_MAINNET,
};

export const alchemy = new Alchemy({
  apiKey: process.env.ALCHEMY_API_KEY,
  network: networkMap[process.env.ALCHEMY_NETWORK],
});
```

### 5. Мониторинг использования

```javascript
// Регулярная проверка лимитов через Dashboard API
// Настройка алертов при достижении 80% лимита
// Использование аналитики для оптимизации запросов
```

## Полезные ссылки

- [Официальная документация](https://docs.alchemy.com/)
- [Alchemy SDK на GitHub](https://github.com/alchemyplatform/alchemy-sdk-js)
- [Alchemy University](https://university.alchemy.com/) — бесплатные курсы
- [Alchemy Dashboard](https://dashboard.alchemy.com/)
- [Status Page](https://status.alchemy.com/)
