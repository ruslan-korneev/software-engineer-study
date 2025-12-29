# Polygon

## Введение

**Polygon** (ранее Matic Network) - это платформа масштабирования Ethereum, которая предоставляет решения для создания быстрых и дешевых транзакций, сохраняя при этом безопасность Ethereum.

## История

### Хронология развития

```
┌─────────────────────────────────────────────────────────────┐
│                    Polygon Timeline                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  2017         2019         2021         2023        2024    │
│   │            │            │            │            │      │
│   ▼            ▼            ▼            ▼            ▼      │
│ ┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────┐     │
│ │Matic│      │Mainnet│   │Rebrand│   │zkEVM │    │POL  │     │
│ │Found│      │Launch │   │Polygon│   │Launch│    │Token│     │
│ └────┘      └────┘      └────┘      └────┘      └────┘     │
│                                                              │
│ Jaynti       Matic        Matic →     Zero       MATIC →    │
│ Kanani,      Network      Polygon     Knowledge  POL        │
│ Sandeep      v1.0         rebrand     EVM        migration  │
│ Nailwal                                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые события

| Дата | Событие |
|------|---------|
| 2017 | Основание Matic Network в Индии |
| Апрель 2019 | IEO на Binance Launchpad |
| Май 2020 | Запуск Matic mainnet |
| Февраль 2021 | Ребрендинг в Polygon |
| Декабрь 2021 | Приобретение Hermez (zkRollup) |
| Март 2023 | Запуск Polygon zkEVM mainnet beta |
| Сентябрь 2024 | Миграция MATIC → POL |

### Основатели

- **Jaynti Kanani** - CEO, бывший разработчик в Housing.com
- **Sandeep Nailwal** - Co-founder, предприниматель
- **Anurag Arjun** - Co-founder, продуктовый менеджер
- **Mihailo Bjelic** - Co-founder, инженер

## Архитектура Polygon

### Polygon 2.0 Vision

```
┌─────────────────────────────────────────────────────────────┐
│                   Polygon 2.0 Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Interoperability Layer                 │    │
│  │                  (AggLayer)                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│           ┌───────────────┼───────────────┐                 │
│           │               │               │                 │
│           ▼               ▼               ▼                 │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │ Polygon  │    │ Polygon  │    │ Polygon  │            │
│    │   PoS    │    │  zkEVM   │    │  CDK     │            │
│    │          │    │          │    │ Chains   │            │
│    └──────────┘    └──────────┘    └──────────┘            │
│           │               │               │                 │
│           └───────────────┼───────────────┘                 │
│                           │                                 │
│                           ▼                                 │
│    ┌─────────────────────────────────────────────────────┐  │
│    │                  Ethereum L1                         │  │
│    │              (Settlement & Data)                     │  │
│    └─────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Типы сетей Polygon

| Сеть | Тип | Описание |
|------|-----|----------|
| **Polygon PoS** | Sidechain | Основная сеть с низкими комиссиями |
| **Polygon zkEVM** | zkRollup | L2 с полной EVM эквивалентностью |
| **Polygon CDK** | Framework | Инструмент для создания своих L2 |
| **Polygon Miden** | zkRollup | Будущий zkRollup на основе STARK |

## Polygon PoS

### Технические характеристики

```
┌─────────────────────────────────────────────────────────────┐
│                   Polygon PoS Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 Polygon PoS Chain                    │   │
│   │                                                      │   │
│   │  Block Time: ~2 секунды                             │   │
│   │  TPS: ~7000                                          │   │
│   │  Validators: 100+                                    │   │
│   │  Consensus: PoS + Bor (block production)            │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                    Checkpoints                               │
│                    (периодически)                            │
│                           │                                  │
│                           ▼                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                  Ethereum Mainnet                    │   │
│   │                                                      │   │
│   │  • Heimdall Layer (валидация checkpoints)           │   │
│   │  • Staking contracts                                 │   │
│   │  • Root chain bridges                                │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Параметры сети

| Параметр | Polygon PoS | Ethereum |
|----------|-------------|----------|
| Block Time | ~2 сек | ~12 сек |
| TPS | ~7000 | ~15-30 |
| Avg Gas Fee | $0.001-0.1 | $1-50 |
| Finality | ~2 мин | ~13 мин |
| Native Token | MATIC/POL | ETH |
| Chain ID | 137 | 1 |

### Консенсус: PoS + Bor

```
┌─────────────────────────────────────────────────────────────┐
│                  Polygon Consensus Layers                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Heimdall                          │    │
│  │            (Proof of Stake + Tendermint)            │    │
│  │                                                      │    │
│  │  • Управление валидаторами                          │    │
│  │  • Checkpoint submission на Ethereum                │    │
│  │  • State sync между сетями                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                      Bor                             │    │
│  │               (Block Production)                     │    │
│  │                                                      │    │
│  │  • Производство блоков                              │    │
│  │  • Выбор producer по очереди (span)                 │    │
│  │  • Форк Geth (go-ethereum)                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Polygon zkEVM

### Что такое zkEVM?

**zkEVM (Zero-Knowledge Ethereum Virtual Machine)** - это L2 решение, которое использует криптографические доказательства (ZK proofs) для валидации транзакций, сохраняя полную совместимость с EVM.

```
┌─────────────────────────────────────────────────────────────┐
│                   Polygon zkEVM Flow                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   User TX → Sequencer → Prover → L1 Contract                │
│                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│   │   User   │ →  │Sequencer │ →  │ Prover   │              │
│   │   TXs    │    │ (batch)  │    │(ZK Proof)│              │
│   └──────────┘    └──────────┘    └──────────┘              │
│                                         │                    │
│                                         ▼                    │
│                    ┌─────────────────────────────────────┐   │
│                    │         Ethereum L1                  │   │
│                    │                                      │   │
│                    │  • Verify ZK proof                   │   │
│                    │  • Store state root                  │   │
│                    │  • Instant finality                  │   │
│                    │                                      │   │
│                    └─────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Типы zkEVM эквивалентности

| Тип | Описание | Примеры |
|-----|----------|---------|
| Type 1 | Полная эквивалентность Ethereum | (в разработке) |
| Type 2 | EVM эквивалентность | Polygon zkEVM, Scroll |
| Type 3 | Почти EVM эквивалентность | - |
| Type 4 | Высокоуровневая совместимость | zkSync Era, StarkNet |

Polygon zkEVM относится к **Type 2** - полная EVM эквивалентность на уровне байткода.

## MATIC / POL Token

### Токеномика

```
┌─────────────────────────────────────────────────────────────┐
│                    POL Token Utility                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    POL Token                          │   │
│  │              (Total Supply: 10B)                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │                                  │
│        ┌──────────────────┼──────────────────┐              │
│        │                  │                  │              │
│        ▼                  ▼                  ▼              │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│  │ Staking  │       │   Gas    │       │Governance│        │
│  │          │       │   Fees   │       │          │        │
│  │ Secure   │       │ Pay for  │       │  Vote on │        │
│  │ network  │       │   TXs    │       │ proposals│        │
│  └──────────┘       └──────────┘       └──────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Миграция MATIC → POL

```javascript
// POL - новый токен Polygon 2.0
// Миграция 1:1 с MATIC

const MATIC_ADDRESS = "0x7D1AfA7B718fb893dB30A3aBc0Cfc608AaCfeBB0"; // Ethereum
const POL_ADDRESS = "0x0000000000000000000000000000000000001010"; // Polygon PoS

// Автоматическая миграция на Polygon PoS
// Для Ethereum - через migration contract
```

## Экосистема Polygon

### DeFi протоколы

```
┌─────────────────────────────────────────────────────────────┐
│                  Polygon DeFi Ecosystem                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │     DEXes      │  │    Lending     │  │    Yield       │ │
│  │                │  │                │  │                │ │
│  │ • QuickSwap    │  │ • Aave v3      │  │ • Beefy        │ │
│  │ • Uniswap v3   │  │ • Compound     │  │ • Yearn        │ │
│  │ • SushiSwap    │  │ • QiDAO (Mai)  │  │ • Balancer     │ │
│  │ • Curve        │  │                │  │                │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │   Stablecoins  │  │    Bridges     │  │     NFT        │ │
│  │                │  │                │  │                │ │
│  │ • USDC         │  │ • Polygon      │  │ • OpenSea      │ │
│  │ • USDT         │  │   Bridge       │  │ • Rarible      │ │
│  │ • DAI          │  │ • Stargate     │  │ • Aavegotchi   │ │
│  │ • FRAX         │  │ • Hop          │  │                │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Крупные партнеры

- **Starbucks** - Odyssey программа лояльности
- **Reddit** - Collectible Avatars
- **Nike** - .SWOOSH NFT платформа
- **Disney** - NFT коллекции
- **Meta (Instagram)** - NFT интеграция
- **Adobe** - Content Authenticity Initiative

## Разработка на Polygon

### Подключение к сети

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    polygon: {
      url: "https://polygon-rpc.com",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 137,
      gasPrice: 30000000000, // 30 gwei
    },
    polygonAmoy: {
      url: "https://rpc-amoy.polygon.technology",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 80002,
    },
    polygonZkEVM: {
      url: "https://zkevm-rpc.com",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 1101,
    }
  }
};
```

### Добавление сети в MetaMask

```javascript
// Polygon PoS Mainnet
async function addPolygonMainnet() {
  await window.ethereum.request({
    method: 'wallet_addEthereumChain',
    params: [{
      chainId: '0x89', // 137
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
}

// Polygon zkEVM Mainnet
async function addPolygonZkEVM() {
  await window.ethereum.request({
    method: 'wallet_addEthereumChain',
    params: [{
      chainId: '0x44d', // 1101
      chainName: 'Polygon zkEVM',
      nativeCurrency: {
        name: 'Ethereum',
        symbol: 'ETH',
        decimals: 18
      },
      rpcUrls: ['https://zkevm-rpc.com'],
      blockExplorerUrls: ['https://zkevm.polygonscan.com']
    }]
  });
}
```

### Деплой на Polygon

```javascript
// scripts/deploy-polygon.js
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying with account:", deployer.address);
  console.log("Network:", network.name);

  // Проверяем баланс MATIC
  const balance = await ethers.provider.getBalance(deployer.address);
  console.log("Balance:", ethers.formatEther(balance), "MATIC");

  // Деплоим контракт
  const MyContract = await ethers.getContractFactory("MyContract");
  const contract = await MyContract.deploy();

  await contract.waitForDeployment();

  console.log("Contract deployed to:", await contract.getAddress());

  // Верификация на Polygonscan
  if (network.name === "polygon") {
    console.log("Waiting for confirmations...");
    await contract.deploymentTransaction().wait(6);

    await run("verify:verify", {
      address: await contract.getAddress(),
      constructorArguments: []
    });
  }
}

main().catch(console.error);
```

```bash
# Деплой
npx hardhat run scripts/deploy-polygon.js --network polygon

# Верификация отдельно
npx hardhat verify --network polygon CONTRACT_ADDRESS
```

### Работа с мостами

```javascript
// Пример использования Polygon Bridge SDK
const { POSClient, use } = require("@maticnetwork/maticjs");
const { Web3ClientPlugin } = require("@maticnetwork/maticjs-ethers");

// Инициализация
use(Web3ClientPlugin);

const posClient = new POSClient();
await posClient.init({
  network: "mainnet",
  version: "v1",
  parent: {
    provider: ethereumProvider,
    defaultConfig: { from: userAddress }
  },
  child: {
    provider: polygonProvider,
    defaultConfig: { from: userAddress }
  }
});

// Депозит ETH из Ethereum в Polygon
const depositTx = await posClient.depositEther(
  ethers.parseEther("0.1"), // amount
  userAddress // recipient
);
console.log("Deposit tx:", depositTx.transactionHash);

// Вывод из Polygon в Ethereum (занимает ~3 часа)
const withdrawTx = await posClient.exitERC20(
  burnTxHash, // hash транзакции сжигания на Polygon
  { gasLimit: 500000 }
);
```

## Gas оптимизация на Polygon

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @notice На Polygon gas дешевый, но оптимизация все еще важна
 * для UX (скорость) и при высоком объеме транзакций
 */
contract PolygonOptimized {
    // Используем events для дешевого хранения данных
    event DataStored(address indexed user, bytes32 dataHash, uint256 timestamp);

    // Batch операции для экономии base gas
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        require(recipients.length == amounts.length, "Length mismatch");

        for (uint256 i = 0; i < recipients.length; i++) {
            // transfer logic
        }
    }

    // Используем calldata вместо memory для read-only массивов
    function processData(bytes calldata data) external pure returns (bytes32) {
        return keccak256(data);
    }
}
```

## Сравнение Polygon PoS vs zkEVM

| Характеристика | Polygon PoS | Polygon zkEVM |
|----------------|-------------|---------------|
| **Тип** | Sidechain | zkRollup (L2) |
| **Безопасность** | Свои валидаторы | Наследует от Ethereum |
| **Finality** | ~2 минуты | ~10-30 минут |
| **Gas токен** | MATIC/POL | ETH |
| **TPS** | ~7000 | ~2000 |
| **Gas cost** | Очень низкий | Низкий |
| **EVM совместимость** | Полная | Полная (Type 2) |

## Плюсы и минусы

### Преимущества Polygon

| Плюс | Описание |
|------|----------|
| **Низкие комиссии** | Транзакции за доли цента |
| **Высокая скорость** | ~2 сек block time |
| **EVM совместимость** | Легкий портинг из Ethereum |
| **Экосистема** | Богатая экосистема DeFi и NFT |
| **Партнерства** | Крупнейшие бренды Web2 |
| **Множество решений** | PoS, zkEVM, CDK |

### Недостатки

| Минус | Описание |
|-------|----------|
| **Централизация** | ~100 валидаторов в PoS |
| **Зависимость от Ethereum** | Для checkpoints и мостов |
| **Безопасность PoS** | Ниже чем у L1 Ethereum |
| **Сложность экосистемы** | Много разных продуктов |

## Полезные ресурсы

- [polygon.technology](https://polygon.technology) - официальный сайт
- [wiki.polygon.technology](https://wiki.polygon.technology) - документация
- [Polygonscan](https://polygonscan.com) - блок-эксплорер PoS
- [zkEVM Polygonscan](https://zkevm.polygonscan.com) - блок-эксплорер zkEVM
- [Polygon Bridge](https://portal.polygon.technology) - официальный мост
- [Faucet](https://faucet.polygon.technology) - тестовые токены
