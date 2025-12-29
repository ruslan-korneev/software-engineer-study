# Arbitrum

## История

**Arbitrum** — это Layer 2 решение для Ethereum, разработанное компанией **Offchain Labs**. Компания была основана в 2018 году Эдом Фелтеном (бывший технический директор Белого дома), Стивеном Голдфедером и Гарри Калодером.

### Ключевые даты:

| Дата | Событие |
|------|---------|
| 2018 | Основание Offchain Labs |
| Август 2021 | Запуск Arbitrum One (mainnet) |
| Август 2022 | Обновление Nitro |
| Март 2023 | Airdrop токена ARB |
| 2023 | Запуск Arbitrum Orbit |
| 2024 | Запуск Arbitrum Stylus |

## Техническая архитектура

Arbitrum использует технологию **Optimistic Rollup** с уникальными оптимизациями.

### Как работает Arbitrum

```
┌─────────────────────────────────────────────────────┐
│                    Arbitrum One                      │
│                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────┐  │
│  │  Sequencer  │───▶│   Nitro    │───▶│  Batch  │  │
│  │ (Секвенсор) │    │   Engine    │    │ Poster  │  │
│  └─────────────┘    └─────────────┘    └────┬────┘  │
│                                              │       │
└──────────────────────────────────────────────┼───────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────┐
│                    Ethereum L1                       │
│                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────┐  │
│  │   Inbox     │    │  Rollup    │    │ Outbox  │  │
│  │  Contract   │    │  Contract   │    │Contract │  │
│  └─────────────┘    └─────────────┘    └─────────┘  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Ключевые компоненты:

1. **Sequencer** — принимает транзакции и определяет их порядок
2. **Nitro** — оптимизированная среда выполнения на базе Geth
3. **Fraud Proofs** — механизм оспаривания некорректных состояний
4. **AnyTrust** — опциональный режим для сниженных комиссий

### Arbitrum Nitro

**Nitro** — это обновление 2022 года, которое значительно улучшило производительность:

```javascript
// Nitro использует WASM для fraud proofs
// Это позволяет эффективно доказывать корректность выполнения

// Пример структуры Nitro
const nitroArchitecture = {
    execution: {
        engine: 'Geth (модифицированный)',
        compilation: 'Go → WASM',
        proofSystem: 'Interactive Fraud Proofs',
    },
    compression: {
        calldata: 'Brotli compression',
        savings: '~90% vs raw data',
    },
    gasModel: {
        l2Gas: 'ArbGas',
        l1DataCost: 'Динамический, зависит от L1',
    },
};
```

### Fraud Proofs (Доказательства мошенничества)

Arbitrum использует **интерактивные fraud proofs**:

```solidity
// Упрощённая логика challenge протокола
contract RollupCore {
    uint256 public constant CHALLENGE_PERIOD = 7 days;

    struct Assertion {
        bytes32 stateHash;
        uint256 deadline;
        address asserter;
    }

    // Создание assertion
    function stakeOnNewAssertion(
        bytes32 expectedHash,
        uint256 numBlocks
    ) external payable {
        require(msg.value >= STAKE_AMOUNT, "Insufficient stake");

        // Создаём новое утверждение о состоянии
        Assertion memory assertion = Assertion({
            stateHash: expectedHash,
            deadline: block.timestamp + CHALLENGE_PERIOD,
            asserter: msg.sender
        });

        // Сохраняем assertion
        assertions[assertionCount++] = assertion;
    }

    // Начало challenge
    function createChallenge(
        uint256 assertionId,
        bytes32 claimedStateHash
    ) external payable returns (uint256 challengeId) {
        Assertion storage assertion = assertions[assertionId];
        require(block.timestamp < assertion.deadline, "Too late");
        require(msg.value >= STAKE_AMOUNT, "Stake required");

        // Начинаем интерактивный процесс bisection
        return startBisection(assertionId, claimedStateHash);
    }
}
```

## Сети Arbitrum

### Arbitrum One

**Основная сеть** для DeFi и общего использования:

| Параметр | Значение |
|----------|----------|
| Chain ID | 42161 |
| Native Token | ETH |
| TPS | ~4,000 |
| Block time | ~0.25 сек |
| Finality | ~7 дней (L1) |

### Arbitrum Nova

**Сеть для игр и социальных приложений** с технологией AnyTrust:

```javascript
// AnyTrust использует Data Availability Committee (DAC)
// Вместо публикации всех данных на L1

const novaConfig = {
    chainId: 42170,
    dataAvailability: 'AnyTrust DAC',
    committee: [
        'Offchain Labs',
        'Google Cloud',
        'Consensys',
        'QuickNode',
        // + другие члены
    ],
    trustAssumption: '2 из N честных членов',
    gasCost: 'Значительно ниже Arbitrum One',
};
```

| Параметр | Arbitrum One | Arbitrum Nova |
|----------|--------------|---------------|
| DA | Ethereum L1 | AnyTrust DAC |
| Безопасность | Максимальная | Высокая |
| Стоимость | $0.10-0.50 | $0.001-0.01 |
| Use case | DeFi, NFT | Gaming, Social |

### Arbitrum Orbit

**Orbit** — это фреймворк для создания собственных L3 на базе Arbitrum:

```javascript
// Orbit позволяет создавать кастомные цепи
const orbitChainConfig = {
    parent: 'Arbitrum One | Nova | Ethereum',
    consensus: 'Rollup | AnyTrust',
    customizations: {
        gasToken: 'Native token (не обязательно ETH)',
        permissions: 'Permissioned или Permissionless',
        governance: 'Custom governance',
    },
    examples: [
        'XAI (gaming)',
        'Rari Chain (NFT)',
        'Degen Chain',
    ],
};
```

## Токен ARB

### Tokenomics

```javascript
const arbTokenomics = {
    totalSupply: '10,000,000,000 ARB',
    distribution: {
        community: '42.78%',      // Airdrop + DAO Treasury
        investors: '17.53%',       // Offchain Labs investors
        team: '26.94%',            // Offchain Labs team
        advisors: '0.75%',         // Advisors
        daoTreasury: '12%',        // DAO Treasury
    },
    vesting: {
        team: '4 года, 1 год cliff',
        investors: '4 года',
    },
    utility: [
        'Governance голосование',
        'Делегирование',
        'Оплата сервисов DAO',
    ],
};
```

### Governance

Arbitrum управляется **DAO** через систему голосования:

```solidity
// Пример governance предложения
interface IArbitrumGovernor {
    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) external returns (uint256 proposalId);

    function castVote(
        uint256 proposalId,
        uint8 support  // 0 = против, 1 = за, 2 = воздержался
    ) external returns (uint256 weight);

    function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) external payable returns (uint256 proposalId);
}
```

## Экосистема Arbitrum

### Ключевые протоколы

| Категория | Протокол | TVL | Описание |
|-----------|----------|-----|----------|
| DEX | Uniswap | $500M+ | Главный DEX |
| Perps | GMX | $500M+ | Децентрализованные perpetuals |
| Lending | Radiant | $200M+ | Cross-chain lending |
| DEX | Camelot | $100M+ | Native DEX с NFT |
| Yield | Pendle | $300M+ | Yield trading |
| Stablecoin | Frax | $100M+ | Stablecoin экосистема |

### Пример взаимодействия с GMX

```javascript
const { ethers } = require('ethers');

// GMX Router для свопов
const GMX_ROUTER = '0xaBBc5F99639c9B6bCb58544ddf04EFA6802F4064';

const routerABI = [
    'function swap(address[] _path, uint256 _amountIn, uint256 _minOut, address _receiver) external',
    'function createIncreasePosition(address[] _path, address _indexToken, uint256 _amountIn, uint256 _minOut, uint256 _sizeDelta, bool _isLong, uint256 _acceptablePrice) external payable',
];

async function swapOnGMX(provider, signer, tokenIn, tokenOut, amount) {
    const router = new ethers.Contract(GMX_ROUTER, routerABI, signer);

    const path = [tokenIn, tokenOut];
    const minOut = 0; // В продакшене рассчитывайте slippage!

    const tx = await router.swap(
        path,
        amount,
        minOut,
        signer.address
    );

    return tx.wait();
}
```

## Разработка на Arbitrum

### Идентичность с Ethereum

Arbitrum **полностью совместим с EVM**, поэтому используются те же инструменты:

```bash
# Установка Hardhat
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Для Arbitrum-специфичных функций
npm install @arbitrum/sdk
```

### Hardhat конфигурация

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
    solidity: "0.8.20",
    networks: {
        arbitrumOne: {
            url: "https://arb1.arbitrum.io/rpc",
            chainId: 42161,
            accounts: [process.env.PRIVATE_KEY],
        },
        arbitrumNova: {
            url: "https://nova.arbitrum.io/rpc",
            chainId: 42170,
            accounts: [process.env.PRIVATE_KEY],
        },
        arbitrumSepolia: {
            url: "https://sepolia-rollup.arbitrum.io/rpc",
            chainId: 421614,
            accounts: [process.env.PRIVATE_KEY],
        },
    },
    etherscan: {
        apiKey: {
            arbitrumOne: process.env.ARBISCAN_API_KEY,
        },
    },
};
```

### Arbitrum Stylus

**Stylus** — революционное обновление, позволяющее писать смарт-контракты на **Rust, C, C++**:

```rust
// Пример Stylus контракта на Rust
#![cfg_attr(not(feature = "std"), no_std)]

extern crate alloc;

use stylus_sdk::{
    alloy_primitives::U256,
    prelude::*,
    storage::StorageU256,
};

#[storage]
#[entrypoint]
pub struct Counter {
    count: StorageU256,
}

#[public]
impl Counter {
    pub fn get(&self) -> U256 {
        self.count.get()
    }

    pub fn increment(&mut self) {
        let current = self.count.get();
        self.count.set(current + U256::from(1));
    }

    pub fn set(&mut self, value: U256) {
        self.count.set(value);
    }
}
```

**Преимущества Stylus:**
- Производительность: 10-100x быстрее Solidity для вычислений
- Язык: Rust, C, C++ (любой язык → WASM)
- Совместимость: Stylus контракты могут вызывать Solidity контракты и наоборот
- Memory-safe: Преимущества Rust для безопасности

### Работа с Arbitrum SDK

```javascript
const { L1ToL2MessageGasEstimator } = require('@arbitrum/sdk');
const { ethers } = require('ethers');

// Бридж ETH на Arbitrum
async function depositEthToArbitrum(l1Signer, amount) {
    const {
        EthBridger,
        getL2Network
    } = require('@arbitrum/sdk');

    const l2Network = await getL2Network(42161); // Arbitrum One
    const ethBridger = new EthBridger(l2Network);

    const depositTx = await ethBridger.deposit({
        amount: ethers.parseEther(amount.toString()),
        l1Signer,
    });

    console.log(`Deposit tx: ${depositTx.hash}`);

    const depositReceipt = await depositTx.wait();
    console.log('Deposit confirmed on L1');

    // Ожидание L2 транзакции
    const l2Result = await depositReceipt.waitForL2(l2Provider);
    console.log(`L2 tx: ${l2Result.transactionHash}`);
}

// Вывод ETH на L1 (занимает ~7 дней)
async function withdrawEthToL1(l2Signer, amount) {
    const { EthBridger, getL2Network } = require('@arbitrum/sdk');

    const l2Network = await getL2Network(42161);
    const ethBridger = new EthBridger(l2Network);

    const withdrawTx = await ethBridger.withdraw({
        amount: ethers.parseEther(amount.toString()),
        l2Signer,
        destinationAddress: await l2Signer.getAddress(),
    });

    const withdrawReceipt = await withdrawTx.wait();
    console.log('Withdrawal initiated');
    console.log('Wait ~7 days for challenge period...');

    // После challenge period можно выполнить claim на L1
}
```

## Плюсы и минусы

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Низкие комиссии** | $0.10-0.50 за транзакцию |
| **Высокая TPS** | ~4,000 транзакций в секунду |
| **EVM совместимость** | Используй те же инструменты |
| **Nitro** | Оптимизированная производительность |
| **Stylus** | Rust/C/C++ контракты |
| **Экосистема** | Крупнейшая L2 экосистема |
| **Децентрализация** | DAO governance |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **7 дней вывода** | Долгий период оспаривания |
| **Централизованный секвенсор** | Пока единственный секвенсор |
| **Зависимость от L1** | При проблемах Ethereum страдает и Arbitrum |
| **Fraud proof редки** | Система редко используется |

## Сравнение с конкурентами

| Параметр | Arbitrum | Optimism | zkSync |
|----------|----------|----------|--------|
| Тип | Optimistic | Optimistic | ZK |
| TVL | $10B+ | $7B+ | $1B+ |
| TPS | ~4,000 | ~2,000 | ~2,000 |
| Финальность | 7 дней | 7 дней | Часы |
| Stylus/Cairo | Stylus (Rust) | - | - |
| Экосистема | Крупнейшая | OP Stack | Развивается |

## Ресурсы

- [Arbitrum Docs](https://docs.arbitrum.io/) — официальная документация
- [Arbiscan](https://arbiscan.io/) — блок-эксплорер
- [Arbitrum Bridge](https://bridge.arbitrum.io/) — официальный мост
- [Arbitrum SDK](https://github.com/OffchainLabs/arbitrum-sdk) — SDK для разработчиков
- [Stylus Docs](https://docs.arbitrum.io/stylus/stylus-gentle-introduction) — документация Stylus
