# Optimism

## История

**Optimism** — это Layer 2 решение для Ethereum, разработанное компанией **Optimism PBC** (ранее Plasma Group). Проект был основан Джингланом Вангом, Карлом Флоершем и Кевином Хо в 2019 году.

### Ключевые даты:

| Дата | Событие |
|------|---------|
| 2019 | Основание Optimism PBC |
| Январь 2021 | Запуск Optimism (alpha) |
| Декабрь 2021 | Публичный mainnet |
| Май 2022 | Airdrop токена OP |
| Июнь 2023 | Запуск Bedrock upgrade |
| 2023-2024 | Развитие OP Stack и Superchain |

## Техническая архитектура

Optimism использует **Optimistic Rollup** с фокусом на **EVM Equivalence** (эквивалентность EVM).

### EVM Equivalence vs EVM Compatibility

```
EVM Compatibility (Arbitrum):
├── Поддержка Solidity
├── Модифицированные opcodes
└── Некоторые отличия в поведении

EVM Equivalence (Optimism):
├── Идентичный bytecode
├── Идентичные opcodes
├── Идентичное поведение газа
└── Минимальные отличия от Ethereum
```

Это означает, что **любой код Ethereum работает на Optimism без изменений**.

### Архитектура Bedrock

**Bedrock** — это обновление 2023 года, которое значительно улучшило архитектуру:

```
┌─────────────────────────────────────────────────────────┐
│                    Optimism Bedrock                      │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  Sequencer  │───▶│ op-node    │───▶│   op-geth   │  │
│  │             │    │ (Rollup)    │    │ (Execution) │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
│         │                                     │          │
│         │              ┌──────────────────────┘          │
│         │              │                                 │
│         ▼              ▼                                 │
│  ┌─────────────────────────────┐                        │
│  │       Batch Submitter       │                        │
│  └──────────────┬──────────────┘                        │
│                 │                                        │
└─────────────────┼────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│                     Ethereum L1                          │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ OptimismPortal│  │L1CrossDomain│   │L2OutputOracle│ │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Ключевые компоненты:

```javascript
const optimismComponents = {
    // Выполнение транзакций
    'op-geth': {
        description: 'Модифицированный Geth для L2',
        role: 'Execution Layer',
        derivation: 'Minimal diff от Ethereum Geth',
    },

    // Rollup логика
    'op-node': {
        description: 'Rollup consensus client',
        role: 'Consensus Layer',
        functions: ['Derivation', 'Sequencing', 'L1 sync'],
    },

    // Секвенсор
    'op-batcher': {
        description: 'Отправка batch на L1',
        role: 'Data Availability',
        compression: 'Сжатие calldata',
    },

    // Proposer
    'op-proposer': {
        description: 'Публикация state roots',
        role: 'State commitments',
        interval: 'Каждые ~2 часа',
    },
};
```

### Fraud Proofs (Cannon)

Optimism использует систему **Cannon** для fraud proofs:

```solidity
// Упрощённая логика fault dispute game
contract FaultDisputeGame {
    // Состояние игры
    enum GameStatus { IN_PROGRESS, CHALLENGER_WIN, DEFENDER_WIN }

    struct Claim {
        bytes32 stateHash;
        uint128 position;   // Позиция в дереве
        address claimant;
    }

    Claim[] public claims;
    GameStatus public status;

    // Инициализация challenge
    function attack(uint256 parentIndex, bytes32 claimHash) external {
        // Атака на позицию оппонента
        Claim memory parent = claims[parentIndex];

        claims.push(Claim({
            stateHash: claimHash,
            position: parent.position * 2,  // Двоичный поиск
            claimant: msg.sender
        }));
    }

    function defend(uint256 parentIndex, bytes32 claimHash) external {
        // Защита позиции
        Claim memory parent = claims[parentIndex];

        claims.push(Claim({
            stateHash: claimHash,
            position: parent.position * 2 + 1,
            claimant: msg.sender
        }));
    }

    // Финальный шаг — исполнение одной инструкции
    function step(
        uint256 claimIndex,
        bytes calldata preimage,
        bytes calldata proof
    ) external {
        // MIPS VM выполняет одну инструкцию
        // и проверяет корректность перехода состояния
    }
}
```

## OP Stack

**OP Stack** — это модульный фреймворк для создания собственных L2 на базе Optimism.

```
┌─────────────────────────────────────────────────────────┐
│                       OP Stack                           │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │ Governance  │  │ Settlement  │  │ Data Availability│ │
│  │   Layer     │  │   Layer     │  │      Layer       │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  Execution  │  │  Sequencing │  │   Derivation    │  │
│  │   Layer     │  │    Layer    │  │     Layer       │  │
│  └─────────────┘  └─────────────┘  └─────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Модули OP Stack:

```javascript
const opStackModules = {
    // Data Availability
    dataAvailability: {
        options: ['Ethereum', 'Celestia', 'EigenDA', 'Custom'],
        default: 'Ethereum calldata',
    },

    // Execution
    execution: {
        options: ['op-geth', 'op-reth', 'Custom EVM'],
        default: 'op-geth (EVM Equivalent)',
    },

    // Sequencing
    sequencing: {
        options: ['Centralized', 'Shared Sequencer', 'Based'],
        default: 'Centralized sequencer',
    },

    // Settlement
    settlement: {
        options: ['Ethereum', 'Other L1'],
        default: 'Ethereum mainnet',
    },
};
```

### Сети на OP Stack

| Сеть | Оператор | Особенности |
|------|----------|-------------|
| **Optimism** | Optimism Foundation | Основная сеть |
| **Base** | Coinbase | Крупнейшая по объёму |
| **Zora** | Zora | NFT-ориентированная |
| **Mode** | Mode Network | DeFi-focused |
| **Fraxtal** | Frax Finance | Frax экосистема |
| **Worldcoin** | Worldchain | World ID |

## Superchain Vision

**Superchain** — это видение экосистемы связанных L2 на базе OP Stack:

```
┌─────────────────────────────────────────────────────────────┐
│                         Superchain                           │
│                                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ Optimism │  │   Base   │  │   Zora   │  │   Mode   │   │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│        │             │             │             │          │
│        └─────────────┼─────────────┼─────────────┘          │
│                      │             │                         │
│              ┌───────▼─────────────▼───────┐                │
│              │   Shared Message Passing    │                │
│              │   Shared Sequencing (WIP)   │                │
│              │   Shared Liquidity (WIP)    │                │
│              └─────────────────────────────┘                │
│                            │                                 │
└────────────────────────────┼─────────────────────────────────┘
                             ▼
                    ┌─────────────────┐
                    │   Ethereum L1   │
                    └─────────────────┘
```

### Преимущества Superchain:

```javascript
const superchainBenefits = {
    // Общая безопасность
    sharedSecurity: {
        description: 'Все сети используют Ethereum L1',
        benefit: 'Единый trust assumption',
    },

    // Интероперабельность
    interoperability: {
        description: 'Нативный cross-chain messaging',
        benefit: 'Быстрые переводы между сетями',
    },

    // Shared upgrades
    sharedUpgrades: {
        description: 'Единые обновления протокола',
        benefit: 'Снижение fragmentation',
    },

    // Liquidity
    sharedLiquidity: {
        description: 'Общая ликвидность (будущее)',
        benefit: 'Лучший UX',
    },
};
```

## Токен OP

### Tokenomics

```javascript
const opTokenomics = {
    totalSupply: '4,294,967,296 OP', // 2^32
    initialSupply: '4,294,967,296 OP',
    inflation: '2% годовых (начиная с 2024)',

    distribution: {
        ecosystem: '25%',        // Ecosystem Fund
        retroPGF: '20%',         // Retroactive Public Goods
        airdrops: '19%',         // Multiple airdrops
        coreContributors: '19%', // Team
        investors: '17%',        // Investors
    },

    vesting: {
        team: '4 года, с cliff',
        investors: '4 года',
    },
};
```

### Retroactive Public Goods Funding (RetroPGF)

**RetroPGF** — уникальная программа Optimism для финансирования общественных благ:

```javascript
const retroPGF = {
    concept: 'Награждение за уже созданную ценность',

    rounds: [
        {
            round: 1,
            amount: '$1M',
            recipients: 58,
            focus: 'L2 инфраструктура',
        },
        {
            round: 2,
            amount: '$10M',
            recipients: 195,
            focus: 'OP Stack, образование, инструменты',
        },
        {
            round: 3,
            amount: '$30M',
            recipients: 501,
            focus: 'Разработчики, создатели контента',
        },
        {
            round: 4,
            amount: '$10M',
            recipients: 207,
            focus: 'Onchain builders',
        },
    ],

    mechanism: 'Citizens\' House голосует за распределение',
};
```

### Governance (Bicameral)

Optimism использует **двухпалатную систему управления**:

```
┌─────────────────────────────────────────────────────┐
│              Optimism Collective                     │
│                                                      │
│  ┌───────────────────┐  ┌───────────────────────┐  │
│  │   Token House     │  │   Citizens' House      │  │
│  │                   │  │                        │  │
│  │ • OP holders      │  │ • Soulbound badges     │  │
│  │ • Protocol        │  │ • RetroPGF voting      │  │
│  │   governance      │  │ • Grant allocation     │  │
│  │ • Upgrades        │  │ • Veto power           │  │
│  └───────────────────┘  └───────────────────────┘  │
│                                                      │
└─────────────────────────────────────────────────────┘
```

```solidity
// Пример голосования в Token House
interface IOptimismGovernor {
    function propose(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas,
        string calldata description
    ) external returns (uint256 proposalId);

    function castVoteWithReason(
        uint256 proposalId,
        uint8 support,
        string calldata reason
    ) external returns (uint256 weight);

    // Делегирование голосов
    function delegate(address delegatee) external;
}
```

## Экосистема Optimism

### Ключевые протоколы

| Категория | Протокол | TVL | Описание |
|-----------|----------|-----|----------|
| DEX | Velodrome | $300M+ | Главный DEX, ve(3,3) модель |
| Derivatives | Synthetix | $200M+ | Синтетические активы |
| DEX | Uniswap | $150M+ | Универсальный DEX |
| Lending | Aave | $200M+ | Lending protocol |
| Perps | Kwenta | $50M+ | Synthetix perps frontend |
| Bridge | Across | - | Быстрый cross-chain мост |

### Пример взаимодействия с Velodrome

```javascript
const { ethers } = require('ethers');

// Velodrome Router
const VELODROME_ROUTER = '0xa062aE8A9c5e11aaA026fc2670B0D65cCc8B2858';

const routerABI = [
    'function swapExactTokensForTokens(uint amountIn, uint amountOutMin, tuple(address from, address to, bool stable)[] routes, address to, uint deadline) external returns (uint[] amounts)',
    'function getAmountsOut(uint amountIn, tuple(address from, address to, bool stable)[] routes) view returns (uint[] amounts)',
];

async function swapOnVelodrome(signer, tokenIn, tokenOut, amountIn, stable = false) {
    const router = new ethers.Contract(VELODROME_ROUTER, routerABI, signer);

    const routes = [{
        from: tokenIn,
        to: tokenOut,
        stable: stable, // true для stablecoin пар
    }];

    // Получаем ожидаемый выход
    const amounts = await router.getAmountsOut(amountIn, routes);
    const amountOutMin = amounts[1] * 99n / 100n; // 1% slippage

    const deadline = Math.floor(Date.now() / 1000) + 1800; // 30 минут

    const tx = await router.swapExactTokensForTokens(
        amountIn,
        amountOutMin,
        routes,
        signer.address,
        deadline
    );

    return tx.wait();
}
```

## Base (Coinbase L2)

**Base** — это L2 от Coinbase, построенный на OP Stack:

```javascript
const baseInfo = {
    operator: 'Coinbase',
    launched: 'Август 2023',
    chainId: 8453,

    features: {
        opStack: 'Полностью на OP Stack',
        noToken: 'Нет нативного токена',
        fees: 'Часть fees → Optimism Collective',
        sequencer: 'Управляется Coinbase',
    },

    advantages: {
        coinbaseIntegration: 'Прямой onramp из Coinbase',
        userBase: 'Доступ к 100M+ пользователей Coinbase',
        liquidity: 'Coinbase Prime интеграция',
        compliance: 'Фокус на соответствие регуляциям',
    },

    ecosystem: [
        'Aerodrome (DEX)',
        'friend.tech',
        'Farcaster apps',
    ],
};
```

### Подключение к Base

```javascript
// hardhat.config.js
module.exports = {
    networks: {
        base: {
            url: 'https://mainnet.base.org',
            chainId: 8453,
            accounts: [process.env.PRIVATE_KEY],
        },
        baseSepolia: {
            url: 'https://sepolia.base.org',
            chainId: 84532,
            accounts: [process.env.PRIVATE_KEY],
        },
    },
};
```

## Разработка на Optimism

### EVM Equivalence на практике

Благодаря EVM Equivalence, разработка идентична Ethereum:

```bash
# Стандартные инструменты
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Для L1 <-> L2 messaging
npm install @eth-optimism/sdk
```

### Hardhat конфигурация

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
    solidity: "0.8.20",
    networks: {
        optimism: {
            url: "https://mainnet.optimism.io",
            chainId: 10,
            accounts: [process.env.PRIVATE_KEY],
        },
        optimismSepolia: {
            url: "https://sepolia.optimism.io",
            chainId: 11155420,
            accounts: [process.env.PRIVATE_KEY],
        },
    },
    etherscan: {
        apiKey: {
            optimisticEthereum: process.env.OPTIMISM_ETHERSCAN_KEY,
        },
    },
};
```

### Cross-Domain Messaging

Optimism имеет встроенную систему сообщений между L1 и L2:

```solidity
// Контракт на L1, отправляющий сообщение на L2
import { ICrossDomainMessenger } from "@eth-optimism/contracts/libraries/bridge/ICrossDomainMessenger.sol";

contract L1Sender {
    address public l1CrossDomainMessenger;
    address public l2Target;

    constructor(address _messenger, address _target) {
        l1CrossDomainMessenger = _messenger;
        l2Target = _target;
    }

    function sendMessageToL2(bytes memory message) external {
        ICrossDomainMessenger(l1CrossDomainMessenger).sendMessage(
            l2Target,
            message,
            1000000  // Gas limit на L2
        );
    }
}

// Контракт на L2, получающий сообщение
import { ICrossDomainMessenger } from "@eth-optimism/contracts/libraries/bridge/ICrossDomainMessenger.sol";

contract L2Receiver {
    address public l2CrossDomainMessenger;
    address public l1Sender;

    event MessageReceived(bytes message);

    constructor(address _messenger, address _sender) {
        l2CrossDomainMessenger = _messenger;
        l1Sender = _sender;
    }

    function receiveMessage(bytes memory message) external {
        // Проверяем, что сообщение от правильного L1 контракта
        require(
            msg.sender == l2CrossDomainMessenger,
            "Not from messenger"
        );
        require(
            ICrossDomainMessenger(l2CrossDomainMessenger).xDomainMessageSender() == l1Sender,
            "Wrong sender"
        );

        emit MessageReceived(message);
    }
}
```

### Использование Optimism SDK

```javascript
const optimism = require('@eth-optimism/sdk');
const { ethers } = require('ethers');

async function bridgeETHToOptimism(l1Signer, amount) {
    const l1Provider = l1Signer.provider;
    const l2Provider = new ethers.JsonRpcProvider('https://mainnet.optimism.io');

    const messenger = new optimism.CrossChainMessenger({
        l1ChainId: 1,
        l2ChainId: 10,
        l1SignerOrProvider: l1Signer,
        l2SignerOrProvider: l2Provider,
    });

    // Депозит ETH
    console.log('Depositing ETH to Optimism...');
    const tx = await messenger.depositETH(ethers.parseEther(amount.toString()));
    await tx.wait();
    console.log('Deposit initiated:', tx.hash);

    // Ожидание на L2 (~2-3 минуты)
    await messenger.waitForMessageStatus(
        tx.hash,
        optimism.MessageStatus.RELAYED
    );
    console.log('ETH available on Optimism!');
}

async function withdrawETHToL1(l2Signer, amount) {
    const l1Provider = new ethers.JsonRpcProvider(L1_RPC);
    const l2Provider = l2Signer.provider;

    const messenger = new optimism.CrossChainMessenger({
        l1ChainId: 1,
        l2ChainId: 10,
        l1SignerOrProvider: l1Provider,
        l2SignerOrProvider: l2Signer,
    });

    // Инициация вывода
    console.log('Initiating withdrawal...');
    const tx = await messenger.withdrawETH(ethers.parseEther(amount.toString()));
    await tx.wait();

    console.log('Waiting for state root (~1 hour)...');
    await messenger.waitForMessageStatus(
        tx.hash,
        optimism.MessageStatus.READY_TO_PROVE
    );

    // Prove withdrawal
    const proveTx = await messenger.proveMessage(tx.hash);
    await proveTx.wait();

    console.log('Waiting for challenge period (~7 days)...');
    await messenger.waitForMessageStatus(
        tx.hash,
        optimism.MessageStatus.READY_FOR_RELAY
    );

    // Finalize
    const finalizeTx = await messenger.finalizeMessage(tx.hash);
    await finalizeTx.wait();

    console.log('Withdrawal complete!');
}
```

## Плюсы и минусы

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **EVM Equivalence** | Полная совместимость с Ethereum |
| **OP Stack** | Модульный фреймворк для L2 |
| **Superchain** | Экосистема связанных сетей |
| **RetroPGF** | Уникальная модель финансирования |
| **Governance** | Продвинутая двухпалатная система |
| **Base** | Доступ к Coinbase экосистеме |
| **Низкие комиссии** | $0.01-0.50 за транзакцию |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **7 дней вывода** | Долгий период оспаривания |
| **Меньше TVL** | Меньше чем у Arbitrum |
| **Централизация** | Единый секвенсор |
| **Fraud proofs** | Были запущены позже Arbitrum |

## Сравнение Optimism vs Arbitrum

| Параметр | Optimism | Arbitrum |
|----------|----------|----------|
| EVM | Equivalence | Compatibility |
| Fraud Proofs | Cannon (MIPS) | Nitro (WASM) |
| Stack | OP Stack (modular) | Orbit |
| Governance | Bicameral | DAO |
| Funding | RetroPGF | Grants |
| Экосистема | Superchain | Orbit chains |
| TVL | $7B+ | $10B+ |
| Base/Coinbase | Да | Нет |

## Ресурсы

- [Optimism Docs](https://docs.optimism.io/) — официальная документация
- [OP Stack Docs](https://stack.optimism.io/) — документация OP Stack
- [Optimism Etherscan](https://optimistic.etherscan.io/) — блок-эксплорер
- [Optimism Bridge](https://app.optimism.io/bridge) — официальный мост
- [Optimism SDK](https://sdk.optimism.io/) — SDK для разработчиков
- [RetroPGF](https://retrofunding.optimism.io/) — информация о RetroPGF
- [Superchain](https://www.superchain.eco/) — экосистема Superchain
