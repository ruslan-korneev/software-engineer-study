# Другие EVM сети

## Введение

Помимо Ethereum, Polygon и BNB Chain существует множество других EVM-совместимых блокчейнов, каждый из которых предлагает уникальные решения для масштабирования, скорости и стоимости транзакций.

## Avalanche

### Обзор

**Avalanche** - это высокопроизводительная платформа смарт-контрактов, разработанная компанией Ava Labs. Avalanche использует уникальный консенсус-протокол Snowman и архитектуру подсетей (subnets).

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                  Avalanche Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 Primary Network                      │   │
│   │                                                      │   │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│   │  │  X-Chain   │  │  P-Chain   │  │  C-Chain   │    │   │
│   │  │            │  │            │  │            │    │   │
│   │  │  Exchange  │  │  Platform  │  │ Contracts  │    │   │
│   │  │  (Assets)  │  │  (Staking, │  │   (EVM)    │    │   │
│   │  │            │  │  Subnets)  │  │            │    │   │
│   │  │  DAG-based │  │  Snowman   │  │  Snowman   │    │   │
│   │  └────────────┘  └────────────┘  └────────────┘    │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                           ▼                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                     Subnets                          │   │
│   │                                                      │   │
│   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │   │
│   │  │ DeFi   │  │ Gaming │  │Enterprise│  │ Custom │    │   │
│   │  │ Subnet │  │ Subnet │  │ Subnet  │  │ Subnet │    │   │
│   │  └────────┘  └────────┘  └────────┘  └────────┘    │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Три цепи Avalanche

| Цепь | Назначение | Консенсус |
|------|------------|-----------|
| **X-Chain** | Создание и обмен активами | Avalanche DAG |
| **P-Chain** | Стейкинг, валидаторы, подсети | Snowman |
| **C-Chain** | Смарт-контракты (EVM) | Snowman |

### Технические параметры C-Chain

```javascript
// Avalanche C-Chain параметры
const AVALANCHE_C_CHAIN = {
  chainId: 43114,
  chainIdTestnet: 43113, // Fuji
  blockTime: "<2 seconds",
  finality: "<1 second",
  tps: "~4500",
  gasToken: "AVAX",
  consensus: "Snowman (PoS)",
  validatorMinStake: "2000 AVAX",
  delegatorMinStake: "25 AVAX"
};

// Подключение
const avalancheRPC = {
  mainnet: "https://api.avax.network/ext/bc/C/rpc",
  testnet: "https://api.avax-test.network/ext/bc/C/rpc"
};
```

### Subnets (Подсети)

Subnets - это независимые блокчейны в экосистеме Avalanche.

```
┌─────────────────────────────────────────────────────────────┐
│                    Avalanche Subnets                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐                                         │
│  │ Primary Network │ ← Все валидаторы обязаны участвовать   │
│  └────────────────┘                                         │
│          │                                                   │
│          ├─── Subnet A (Gaming)                             │
│          │    • Свои валидаторы                             │
│          │    • Свой токен для gas                          │
│          │    • Свои правила                                │
│          │                                                   │
│          ├─── Subnet B (Enterprise)                         │
│          │    • Permissioned валидаторы                     │
│          │    • KYC/AML compliance                          │
│          │                                                   │
│          └─── Subnet C (DeFi)                               │
│               • Высокий TPS                                  │
│               • Оптимизированный EVM                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Популярные Subnets

- **DFK Chain** - Defi Kingdoms game
- **Swimmer Network** - Crabada game
- **BEAM** - Merit Circle gaming subnet

## Fantom

### Обзор

**Fantom** - это высокопроизводительный блокчейн, использующий DAG-based консенсус Lachesis. Opera - это EVM-совместимый слой Fantom.

### Архитектура Lachesis

```
┌─────────────────────────────────────────────────────────────┐
│                 Fantom Lachesis Consensus                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   aBFT (Asynchronous Byzantine Fault Tolerance)             │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              DAG (Directed Acyclic Graph)            │   │
│   │                                                      │   │
│   │        ┌───┐     ┌───┐     ┌───┐                    │   │
│   │        │ E1│────→│ E2│────→│ E3│                    │   │
│   │        └───┘     └───┘     └───┘                    │   │
│   │           \         │        /                       │   │
│   │            \        │       /                        │   │
│   │             ↘      ↓      ↙                         │   │
│   │              ┌──────────┐                            │   │
│   │              │  Finality │                           │   │
│   │              │  <1 sec   │                           │   │
│   │              └──────────┘                            │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   Преимущества:                                             │
│   • Мгновенная финальность (<1 сек)                        │
│   • Высокий TPS (теоретически до 300,000)                  │
│   • Низкие комиссии                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Технические параметры

```javascript
// Fantom Opera параметры
const FANTOM_OPERA = {
  chainId: 250,
  chainIdTestnet: 4002,
  blockTime: "<1 second",
  finality: "~1-2 seconds",
  tps: "~10000",
  gasToken: "FTM",
  consensus: "Lachesis (aBFT)",
  minStake: "1 FTM (delegation)",
  validatorMinStake: "500,000 FTM"
};

// RPC endpoints
const fantomRPC = {
  mainnet: "https://rpc.ftm.tools",
  testnet: "https://rpc.testnet.fantom.network"
};
```

### Экосистема Fantom

- **SpookySwap** - главный DEX
- **Geist Finance** - lending protocol
- **Beethoven X** - балансер форк
- **Solidly** (Ve(3,3)) - инновационный DEX от Andre Cronje

## Cronos

### Обзор

**Cronos** - это EVM-совместимый блокчейн от Crypto.com, построенный на Cosmos SDK с использованием Ethermint.

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                   Cronos Architecture                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   Cronos Chain                       │   │
│   │                                                      │   │
│   │  ┌──────────────┐      ┌──────────────┐             │   │
│   │  │   Ethermint  │      │   Cosmos SDK │             │   │
│   │  │     (EVM)    │──────│  (Tendermint)│             │   │
│   │  └──────────────┘      └──────────────┘             │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                    IBC Protocol                              │
│                           │                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                Cosmos Ecosystem                      │   │
│   │                                                      │   │
│   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │   │
│   │  │Crypto  │  │ Cosmos │  │Osmosis │  │  Terra │    │   │
│   │  │.com    │  │  Hub   │  │        │  │        │    │   │
│   │  │Chain   │  │        │  │        │  │        │    │   │
│   │  └────────┘  └────────┘  └────────┘  └────────┘    │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Параметры Cronos

```javascript
// Cronos параметры
const CRONOS = {
  chainId: 25,
  chainIdTestnet: 338,
  blockTime: "~5 seconds",
  gasToken: "CRO",
  consensus: "Tendermint PoS",
  ibcEnabled: true // Inter-Blockchain Communication
};

// RPC
const cronosRPC = {
  mainnet: "https://evm.cronos.org",
  testnet: "https://evm-t3.cronos.org"
};
```

### Особенности

- **IBC** - связь с Cosmos экосистемой
- **Crypto.com** - интеграция с биржей и картами
- **VVS Finance** - главный DEX
- **Tectonic** - lending protocol

## Layer 2 Rollups

### zkSync Era

**zkSync Era** - это zkRollup от Matter Labs с собственным подходом к EVM совместимости.

```
┌─────────────────────────────────────────────────────────────┐
│                    zkSync Era Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   zkSync Era (L2)                    │   │
│   │                                                      │   │
│   │  • Type 4 zkEVM (высокоуровневая совместимость)     │   │
│   │  • Native Account Abstraction                       │   │
│   │  • Paymaster (gas в любых токенах)                 │   │
│   │  • LLVM-based компилятор                           │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                      ZK Proofs                               │
│                           │                                  │
│                           ▼                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                  Ethereum L1                         │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```javascript
// zkSync Era параметры
const ZKSYNC_ERA = {
  chainId: 324,
  chainIdTestnet: 280,
  gasToken: "ETH",
  type: "zkRollup",
  accountAbstraction: true,
  paymasters: true
};

// Особенности zkSync Era
const features = {
  // Native Account Abstraction
  accountAbstraction: `
    // Все аккаунты - смарт-контракты
    // Можно оплачивать gas любыми токенами
    // Кастомная логика валидации
  `,

  // Paymaster
  paymaster: `
    // Спонсирование gas для пользователей
    // Оплата gas в ERC-20 токенах
  `
};
```

### Base

**Base** - это L2 от Coinbase, построенный на OP Stack (Optimism).

```javascript
// Base параметры
const BASE = {
  chainId: 8453,
  chainIdTestnet: 84532, // Sepolia
  gasToken: "ETH",
  type: "Optimistic Rollup",
  stack: "OP Stack",
  operator: "Coinbase"
};

// Особенности Base
const baseFeatures = {
  coinbaseIntegration: "Easy onboarding from Coinbase",
  lowFees: "$0.001-0.10 per transaction",
  developerFriendly: "Full OP Stack compatibility"
};
```

### Linea

**Linea** - это zkEVM L2 от ConsenSys (создатели MetaMask).

```javascript
// Linea параметры
const LINEA = {
  chainId: 59144,
  chainIdTestnet: 59141,
  gasToken: "ETH",
  type: "zkRollup",
  zkType: "Type 2 zkEVM",
  operator: "ConsenSys"
};
```

### Arbitrum

**Arbitrum** - один из крупнейших L2 на Optimistic Rollup технологии.

```javascript
// Arbitrum параметры
const ARBITRUM = {
  arbitrumOne: {
    chainId: 42161,
    type: "Optimistic Rollup",
    gasToken: "ETH"
  },
  arbitrumNova: {
    chainId: 42170,
    type: "AnyTrust (Data Availability Committee)",
    gasToken: "ETH",
    useCase: "Gaming, Social"
  }
};
```

### Optimism

**Optimism** - L2 на Optimistic Rollup, создатели OP Stack.

```javascript
// Optimism параметры
const OPTIMISM = {
  chainId: 10,
  chainIdTestnet: 11155420, // Sepolia
  gasToken: "ETH",
  type: "Optimistic Rollup",
  governanceToken: "OP"
};
```

## Сравнительная таблица

| Сеть | Тип | Chain ID | Консенсус | TPS | Gas Fee | Finality |
|------|-----|----------|-----------|-----|---------|----------|
| **Avalanche C** | L1 | 43114 | Snowman | ~4500 | $0.01-0.5 | <1 сек |
| **Fantom** | L1 | 250 | Lachesis | ~10000 | $0.001-0.01 | <1 сек |
| **Cronos** | L1 | 25 | Tendermint | ~200 | $0.01-0.1 | ~5 сек |
| **zkSync Era** | L2 zkRollup | 324 | ZK Proofs | ~2000 | $0.01-0.5 | мгновенно* |
| **Base** | L2 Optimistic | 8453 | Fraud Proofs | ~2000 | $0.001-0.1 | ~7 дней* |
| **Linea** | L2 zkRollup | 59144 | ZK Proofs | ~2000 | $0.01-0.5 | мгновенно* |
| **Arbitrum** | L2 Optimistic | 42161 | Fraud Proofs | ~4000 | $0.01-0.5 | ~7 дней* |
| **Optimism** | L2 Optimistic | 10 | Fraud Proofs | ~2000 | $0.01-0.5 | ~7 дней* |

*Финальность на L1 для вывода средств. Транзакции подтверждаются мгновенно на L2.

## Подключение к сетям

### Hardhat конфигурация

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    // Avalanche
    avalanche: {
      url: "https://api.avax.network/ext/bc/C/rpc",
      chainId: 43114,
      accounts: [process.env.PRIVATE_KEY]
    },
    avalancheFuji: {
      url: "https://api.avax-test.network/ext/bc/C/rpc",
      chainId: 43113,
      accounts: [process.env.PRIVATE_KEY]
    },

    // Fantom
    fantom: {
      url: "https://rpc.ftm.tools",
      chainId: 250,
      accounts: [process.env.PRIVATE_KEY]
    },

    // Cronos
    cronos: {
      url: "https://evm.cronos.org",
      chainId: 25,
      accounts: [process.env.PRIVATE_KEY]
    },

    // zkSync Era
    zkSyncEra: {
      url: "https://mainnet.era.zksync.io",
      chainId: 324,
      accounts: [process.env.PRIVATE_KEY],
      zksync: true, // требует @matterlabs/hardhat-zksync
    },

    // Base
    base: {
      url: "https://mainnet.base.org",
      chainId: 8453,
      accounts: [process.env.PRIVATE_KEY]
    },

    // Linea
    linea: {
      url: "https://rpc.linea.build",
      chainId: 59144,
      accounts: [process.env.PRIVATE_KEY]
    },

    // Arbitrum
    arbitrum: {
      url: "https://arb1.arbitrum.io/rpc",
      chainId: 42161,
      accounts: [process.env.PRIVATE_KEY]
    },

    // Optimism
    optimism: {
      url: "https://mainnet.optimism.io",
      chainId: 10,
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};
```

### Добавление сетей в MetaMask

```javascript
// Универсальная функция добавления сети
async function addNetwork(networkConfig) {
  try {
    await window.ethereum.request({
      method: 'wallet_addEthereumChain',
      params: [networkConfig]
    });
  } catch (error) {
    console.error('Error adding network:', error);
  }
}

// Конфигурации сетей
const NETWORKS = {
  avalanche: {
    chainId: '0xA86A', // 43114
    chainName: 'Avalanche C-Chain',
    nativeCurrency: { name: 'AVAX', symbol: 'AVAX', decimals: 18 },
    rpcUrls: ['https://api.avax.network/ext/bc/C/rpc'],
    blockExplorerUrls: ['https://snowtrace.io']
  },
  fantom: {
    chainId: '0xFA', // 250
    chainName: 'Fantom Opera',
    nativeCurrency: { name: 'FTM', symbol: 'FTM', decimals: 18 },
    rpcUrls: ['https://rpc.ftm.tools'],
    blockExplorerUrls: ['https://ftmscan.com']
  },
  cronos: {
    chainId: '0x19', // 25
    chainName: 'Cronos',
    nativeCurrency: { name: 'CRO', symbol: 'CRO', decimals: 18 },
    rpcUrls: ['https://evm.cronos.org'],
    blockExplorerUrls: ['https://cronoscan.com']
  },
  zkSyncEra: {
    chainId: '0x144', // 324
    chainName: 'zkSync Era',
    nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://mainnet.era.zksync.io'],
    blockExplorerUrls: ['https://explorer.zksync.io']
  },
  base: {
    chainId: '0x2105', // 8453
    chainName: 'Base',
    nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://mainnet.base.org'],
    blockExplorerUrls: ['https://basescan.org']
  },
  linea: {
    chainId: '0xE708', // 59144
    chainName: 'Linea',
    nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://rpc.linea.build'],
    blockExplorerUrls: ['https://lineascan.build']
  },
  arbitrum: {
    chainId: '0xA4B1', // 42161
    chainName: 'Arbitrum One',
    nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://arb1.arbitrum.io/rpc'],
    blockExplorerUrls: ['https://arbiscan.io']
  },
  optimism: {
    chainId: '0xA', // 10
    chainName: 'Optimism',
    nativeCurrency: { name: 'ETH', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://mainnet.optimism.io'],
    blockExplorerUrls: ['https://optimistic.etherscan.io']
  }
};
```

## Выбор сети для проекта

```
┌─────────────────────────────────────────────────────────────┐
│                   Как выбрать сеть?                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  🎮 Gaming / High TPS:                                       │
│     → Avalanche Subnet, Fantom, Arbitrum Nova               │
│                                                              │
│  💰 DeFi / High Security:                                    │
│     → Ethereum L1, Arbitrum, Optimism, Base                 │
│                                                              │
│  💸 Low Cost / Mass Adoption:                                │
│     → Polygon PoS, Base, zkSync Era                         │
│                                                              │
│  🔐 Maximum Security (zkRollup):                             │
│     → zkSync Era, Linea, Polygon zkEVM, Scroll              │
│                                                              │
│  🌐 Cosmos Interoperability:                                 │
│     → Cronos                                                │
│                                                              │
│  🏢 Enterprise / Custom Rules:                               │
│     → Avalanche Subnet, Polygon CDK                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Полезные ресурсы

### Avalanche
- [avax.network](https://avax.network) - официальный сайт
- [Snowtrace](https://snowtrace.io) - блок-эксплорер
- [Core Wallet](https://core.app) - официальный кошелек

### Fantom
- [fantom.foundation](https://fantom.foundation) - официальный сайт
- [FTMScan](https://ftmscan.com) - блок-эксплорер

### Cronos
- [cronos.org](https://cronos.org) - официальный сайт
- [CronoScan](https://cronoscan.com) - блок-эксплорер

### L2 Rollups
- [zkSync](https://zksync.io) - официальный сайт zkSync Era
- [Base](https://base.org) - официальный сайт Base
- [Linea](https://linea.build) - официальный сайт Linea
- [Arbitrum](https://arbitrum.io) - официальный сайт Arbitrum
- [Optimism](https://optimism.io) - официальный сайт Optimism

### Агрегаторы информации
- [L2Beat](https://l2beat.com) - статистика L2 решений
- [DefiLlama](https://defillama.com) - TVL по сетям
- [Chainlist](https://chainlist.org) - список EVM сетей и RPC
