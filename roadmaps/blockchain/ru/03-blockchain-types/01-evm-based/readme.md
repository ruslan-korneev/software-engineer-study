# EVM-совместимые блокчейны

## Что такое EVM?

**EVM (Ethereum Virtual Machine)** - это виртуальная машина Ethereum, которая является средой выполнения для смарт-контрактов. EVM представляет собой стековую машину Тьюринга с ограниченными ресурсами (gas), которая изолирует выполнение кода от основной системы.

### Архитектура EVM

```
┌─────────────────────────────────────────────────────────────┐
│                    Ethereum Network                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Node 1    │  │   Node 2    │  │   Node N    │         │
│  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │         │
│  │  │  EVM  │  │  │  │  EVM  │  │  │  │  EVM  │  │         │
│  │  └───────┘  │  │  └───────┘  │  │  └───────┘  │         │
│  │     ↓       │  │     ↓       │  │     ↓       │         │
│  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │         │
│  │  │ State │  │  │  │ State │  │  │  │ State │  │         │
│  │  └───────┘  │  │  └───────┘  │  │  └───────┘  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые компоненты EVM

1. **Stack** - стек из 1024 элементов по 256 бит каждый
2. **Memory** - линейная память, расширяемая во время выполнения
3. **Storage** - постоянное хранилище (key-value на 256 бит)
4. **Calldata** - неизменяемые входные данные транзакции
5. **Code** - байткод смарт-контракта

```
┌─────────────────────────────────────────────────────────────┐
│                       EVM Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │    Stack     │    │    Memory    │    │   Storage    │   │
│  │  (1024 x 256)│    │   (volatile) │    │ (persistent) │   │
│  │              │    │              │    │              │   │
│  │   ┌─────┐    │    │  ┌────────┐  │    │  key: value  │   │
│  │   │ ... │    │    │  │  ...   │  │    │  key: value  │   │
│  │   │ el2 │    │    │  │  data  │  │    │  key: value  │   │
│  │   │ el1 │    │    │  │  data  │  │    │     ...      │   │
│  │   │ el0 │    │    │  └────────┘  │    │              │   │
│  │   └─────┘    │    │              │    │              │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Calldata   │    │     Code     │    │ Program      │   │
│  │  (immutable) │    │  (bytecode)  │    │ Counter (PC) │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Преимущества EVM-совместимости

### Для разработчиков

| Преимущество | Описание |
|--------------|----------|
| **Переносимость кода** | Один и тот же Solidity код работает на любой EVM-сети |
| **Единые инструменты** | Hardhat, Foundry, Remix работают везде |
| **Общая экосистема** | OpenZeppelin, Chainlink и другие библиотеки |
| **Опытные кадры** | Большой пул Solidity разработчиков |

### Для пользователей

- Один кошелек (MetaMask) для всех сетей
- Привычный интерфейс взаимодействия
- Легкий переход между сетями (просто сменить RPC)

### Для проектов

- Быстрый мультичейн деплой
- Кросс-чейн мосты и интероперабельность
- Ликвидность из разных сетей

## Популярные EVM-совместимые сети

```
┌─────────────────────────────────────────────────────────────┐
│              EVM-Compatible Blockchains                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│   │ Ethereum │     │  BNB     │     │ Polygon  │            │
│   │   (ETH)  │     │  Chain   │     │  (MATIC) │            │
│   │  mainnet │     │  (BNB)   │     │   PoS    │            │
│   └──────────┘     └──────────┘     └──────────┘            │
│                                                              │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│   │Avalanche │     │  Fantom  │     │  Cronos  │            │
│   │ C-Chain  │     │  Opera   │     │  (CRO)   │            │
│   │  (AVAX)  │     │  (FTM)   │     │          │            │
│   └──────────┘     └──────────┘     └──────────┘            │
│                                                              │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│   │  zkSync  │     │   Base   │     │  Linea   │            │
│   │   Era    │     │ (Coinbase│     │(Consensys│            │
│   │  (ETH)   │     │   L2)    │     │    L2)   │            │
│   └──────────┘     └──────────┘     └──────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Сравнительная таблица

| Сеть | Консенсус | TPS | Avg. Gas Fee | Block Time | TVL (2024) |
|------|-----------|-----|--------------|------------|------------|
| **Ethereum** | PoS | ~30 | $1-50 | 12 сек | ~$50B |
| **Polygon PoS** | PoS | ~7000 | $0.001-0.1 | 2 сек | ~$1B |
| **BNB Chain** | PoSA | ~160 | $0.05-0.5 | 3 сек | ~$5B |
| **Avalanche C** | Snowball | ~4500 | $0.01-0.5 | <1 сек | ~$1B |
| **Fantom** | Lachesis | ~10000 | $0.001-0.01 | <1 сек | ~$100M |
| **Arbitrum** | Optimistic | ~4000 | $0.01-0.5 | <1 сек | ~$3B |
| **Optimism** | Optimistic | ~2000 | $0.01-0.5 | 2 сек | ~$1B |
| **zkSync Era** | zkRollup | ~2000 | $0.01-0.5 | <1 сек | ~$500M |
| **Base** | Optimistic | ~2000 | $0.001-0.1 | 2 сек | ~$2B |

## Как подключиться к EVM сети

### Добавление сети в MetaMask

```javascript
// Пример добавления Polygon в MetaMask программно
async function addPolygonNetwork() {
  try {
    await window.ethereum.request({
      method: 'wallet_addEthereumChain',
      params: [{
        chainId: '0x89', // 137 в hex
        chainName: 'Polygon Mainnet',
        nativeCurrency: {
          name: 'MATIC',
          symbol: 'MATIC',
          decimals: 18
        },
        rpcUrls: ['https://polygon-rpc.com'],
        blockExplorerUrls: ['https://polygonscan.com']
      }]
    });
  } catch (error) {
    console.error('Ошибка добавления сети:', error);
  }
}
```

### Подключение через ethers.js

```javascript
import { ethers } from 'ethers';

// Подключение к разным сетям
const providers = {
  ethereum: new ethers.JsonRpcProvider('https://eth.llamarpc.com'),
  polygon: new ethers.JsonRpcProvider('https://polygon-rpc.com'),
  bsc: new ethers.JsonRpcProvider('https://bsc-dataseed.binance.org'),
  avalanche: new ethers.JsonRpcProvider('https://api.avax.network/ext/bc/C/rpc'),
  fantom: new ethers.JsonRpcProvider('https://rpc.ftm.tools')
};

// Один и тот же контракт на разных сетях
const CONTRACT_ABI = [...]; // ABI одинаковый везде!

const contracts = {
  ethereum: new ethers.Contract(ETH_ADDRESS, CONTRACT_ABI, providers.ethereum),
  polygon: new ethers.Contract(POLY_ADDRESS, CONTRACT_ABI, providers.polygon),
  bsc: new ethers.Contract(BSC_ADDRESS, CONTRACT_ABI, providers.bsc)
};
```

## Chain IDs популярных сетей

```javascript
const CHAIN_IDS = {
  // Mainnet
  ETHEREUM: 1,
  POLYGON: 137,
  BSC: 56,
  AVALANCHE: 43114,
  FANTOM: 250,
  ARBITRUM: 42161,
  OPTIMISM: 10,
  BASE: 8453,
  ZKSYNC_ERA: 324,
  LINEA: 59144,

  // Testnet
  SEPOLIA: 11155111,
  POLYGON_AMOY: 80002,
  BSC_TESTNET: 97,
  AVALANCHE_FUJI: 43113
};
```

## Заключение

EVM-совместимость стала де-факто стандартом в блокчейн-индустрии. Это позволяет:

- **Разработчикам** - писать код один раз и деплоить везде
- **Проектам** - быстро масштабироваться на новые сети
- **Пользователям** - использовать привычные инструменты

В следующих разделах мы подробно рассмотрим каждую из основных EVM-сетей.

## Навигация

- [Ethereum](./01-ethereum/readme.md) - оригинальная EVM сеть
- [Polygon](./02-polygon/readme.md) - масштабируемое L2 решение
- [BNB Chain](./03-bnb-chain/readme.md) - сеть от Binance
- [Другие EVM сети](./04-other-evm/readme.md) - Avalanche, Fantom, zkSync и другие
