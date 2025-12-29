# BNB Chain (BSC)

## Введение

**BNB Chain** (ранее Binance Smart Chain / BSC) - это блокчейн-платформа, созданная криптобиржей Binance для создания децентрализованных приложений с низкими комиссиями и высокой пропускной способностью.

## История

### Хронология

```
┌─────────────────────────────────────────────────────────────┐
│                    BNB Chain Timeline                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  2017         2019         2020         2022         2023   │
│   │            │            │            │            │      │
│   ▼            ▼            ▼            ▼            ▼      │
│ ┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────┐     │
│ │BNB │      │Binance│    │BSC  │      │Rebrand│   │opBNB │    │
│ │ICO │      │Chain │    │Launch│     │to BNB │   │Launch│    │
│ └────┘      └────┘      └────┘      └────┘      └────┘     │
│                                                              │
│ $15M        DEX          EVM         BNB        L2          │
│ raised      focused      compatible  Chain      solution    │
│                          chain       ecosystem              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые события

| Дата | Событие |
|------|---------|
| Июль 2017 | ICO BNB токена ($0.15 за токен) |
| Апрель 2019 | Запуск Binance Chain (DEX) |
| Сентябрь 2020 | Запуск Binance Smart Chain (BSC) |
| Февраль 2022 | Ребрендинг BSC → BNB Chain |
| Июнь 2023 | Запуск opBNB (L2 решение) |
| 2024 | BNB Greenfield (децентрализованное хранилище) |

### О Binance

- **Основатель**: Changpeng Zhao (CZ)
- **Основание**: 2017 год
- **Штаб-квартира**: Нет фиксированной (распределенная компания)
- **Статус**: Крупнейшая криптобиржа по объему торгов

## Архитектура BNB Chain

### Экосистема BNB

```
┌─────────────────────────────────────────────────────────────┐
│                   BNB Chain Ecosystem                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 BNB Beacon Chain                     │    │
│  │              (Governance & Staking)                  │    │
│  │                                                      │    │
│  │  • BNB staking и governance                         │    │
│  │  • Cross-chain communication                        │    │
│  │  • Validator selection                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              BNB Smart Chain (BSC)                   │    │
│  │            (EVM-compatible Smart Contracts)          │    │
│  │                                                      │    │
│  │  • DeFi, NFT, GameFi                                │    │
│  │  • Fast transactions (~3 sec)                       │    │
│  │  • Low fees (~$0.05-0.5)                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                     opBNB                            │    │
│  │              (L2 Optimistic Rollup)                  │    │
│  │                                                      │    │
│  │  • Ultra-low fees (~$0.001)                         │    │
│  │  • High TPS for gaming                              │    │
│  │  • Based on OP Stack                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  BNB Greenfield                      │    │
│  │            (Decentralized Storage)                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Консенсус: Proof of Staked Authority (PoSA)

### Как работает PoSA

```
┌─────────────────────────────────────────────────────────────┐
│              Proof of Staked Authority (PoSA)                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   PoSA = DPoS (Delegated Proof of Stake)                    │
│        + PoA (Proof of Authority)                           │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                Active Validators (21)                │   │
│   │                                                      │   │
│   │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐        │   │
│   │  │ V1│ │ V2│ │ V3│ │ V4│ │ V5│ │...│ │V21│        │   │
│   │  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘        │   │
│   │                                                      │   │
│   │  Выбираются на основе:                              │   │
│   │  • Количества застейканных BNB                      │   │
│   │  • Репутации                                        │   │
│   │  • Uptime                                           │   │
│   │                                                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                  │
│                     Round-robin                              │
│                    block production                          │
│                           │                                  │
│                           ▼                                  │
│                    ┌──────────────┐                          │
│                    │    Block     │                          │
│                    │   every 3s   │                          │
│                    └──────────────┘                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Сравнение консенсусов

| Параметр | PoSA (BNB) | PoS (Ethereum) | PoW (Bitcoin) |
|----------|------------|----------------|---------------|
| Валидаторы | 21 активных | ~900,000 | Неограничено |
| Block Time | ~3 сек | ~12 сек | ~10 мин |
| Финальность | Мгновенная* | ~13 мин | ~60 мин |
| Децентрализация | Низкая | Высокая | Высокая |
| Энергопотребление | Низкое | Низкое | Высокое |

*Финальность на BNB Chain достигается за ~15 блоков (~45 сек)

## Технические характеристики

### Параметры сети

| Параметр | Значение |
|----------|----------|
| Chain ID | 56 (mainnet), 97 (testnet) |
| Block Time | ~3 секунды |
| Gas Limit | 140M gas/block |
| Avg Gas Price | 3-5 gwei |
| TPS | ~160 (теоретически до 2000) |
| Native Token | BNB |
| EVM Version | Shanghai |

### Стоимость операций

```
┌─────────────────────────────────────────────────────────────┐
│             BNB Chain vs Ethereum Gas Costs                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Операция              BNB Chain        Ethereum            │
│  ─────────────────────────────────────────────────────      │
│  ETH/BNB Transfer      ~$0.01           ~$1-5               │
│  ERC-20 Transfer       ~$0.02           ~$2-10              │
│  Uniswap Swap          ~$0.05-0.20      ~$5-50              │
│  NFT Mint              ~$0.10-0.50      ~$10-100            │
│  Contract Deploy       ~$0.50-2.00      ~$50-500            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## BNB Token

### Токеномика

```
┌─────────────────────────────────────────────────────────────┐
│                      BNB Tokenomics                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Initial Supply: 200,000,000 BNB                            │
│  Current Supply: ~145,000,000 BNB (после burns)             │
│  Target Supply: 100,000,000 BNB                             │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  Token Burns                         │    │
│  │                                                      │    │
│  │  • Quarterly Auto-Burn                              │    │
│  │    (на основе цены BNB и количества блоков)        │    │
│  │                                                      │    │
│  │  • Real-Time Burn (BEP-95)                          │    │
│  │    (часть gas fees сжигается)                       │    │
│  │                                                      │    │
│  │  Уже сожжено: >55M BNB                              │    │
│  │                                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Использование BNB

```javascript
// BNB используется для:
const BNB_USE_CASES = {
  // 1. Оплата комиссий на BNB Chain
  gas: "Оплата транзакций на BSC и opBNB",

  // 2. Скидки на Binance
  trading: "До 25% скидки на торговые комиссии",

  // 3. Staking
  staking: "Валидация сети и получение наград",

  // 4. Launchpad
  launchpad: "Участие в IEO на Binance",

  // 5. DeFi
  defi: "Collateral, ликвидность, фарминг"
};
```

## Экосистема BNB Chain

### DeFi протоколы

```
┌─────────────────────────────────────────────────────────────┐
│                 BNB Chain DeFi Ecosystem                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │     DEXes      │  │    Lending     │  │    Yield       │ │
│  │                │  │                │  │                │ │
│  │ • PancakeSwap  │  │ • Venus        │  │ • Alpaca       │ │
│  │ • Biswap       │  │ • Radiant      │  │ • Beefy        │ │
│  │ • DODO         │  │ • Cream        │  │ • Autofarm     │ │
│  │ • Trader Joe   │  │                │  │                │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │  Derivatives   │  │    Bridges     │  │    Gaming      │ │
│  │                │  │                │  │                │ │
│  │ • ApolloX      │  │ • Binance      │  │ • MOBOX        │ │
│  │ • Level        │  │   Bridge       │  │ • BinaryX      │ │
│  │ • GMX (fork)   │  │ • Multichain   │  │ • SecondLive   │ │
│  │                │  │ • Stargate     │  │                │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### PancakeSwap - главный DEX

```solidity
// PancakeSwap Router v3 интерфейс
interface IPancakeRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);
}

// Адреса на BNB Chain
address constant PANCAKE_ROUTER_V3 = 0x13f4EA83D0bd40E75C8222255bc855a974568Dd4;
address constant WBNB = 0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c;
address constant BUSD = 0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56;
address constant CAKE = 0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82;
```

### Venus Protocol - главный Lending

```solidity
// Venus - форк Compound для BNB Chain
interface IVToken {
    function mint(uint mintAmount) external returns (uint);
    function redeem(uint redeemTokens) external returns (uint);
    function borrow(uint borrowAmount) external returns (uint);
    function repayBorrow(uint repayAmount) external returns (uint);

    function balanceOf(address owner) external view returns (uint);
    function exchangeRateCurrent() external returns (uint);
    function borrowBalanceCurrent(address account) external returns (uint);
}

// vTokens на Venus
address constant vBNB = 0xA07c5b74C9B40447a954e1466938b865b6BBea36;
address constant vBUSD = 0x95c78222B3D6e262426483D42CfA53685A67Ab9D;
address constant vUSDT = 0xfD5840Cd36d94D7229439859C0112a4185BC0255;
```

## Разработка на BNB Chain

### Настройка Hardhat

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: "0.8.20",
  networks: {
    bsc: {
      url: "https://bsc-dataseed.binance.org",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 56,
      gasPrice: 3000000000, // 3 gwei
    },
    bscTestnet: {
      url: "https://data-seed-prebsc-1-s1.binance.org:8545",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 97,
      gasPrice: 10000000000, // 10 gwei
    },
    opBNB: {
      url: "https://opbnb-mainnet-rpc.bnbchain.org",
      accounts: [process.env.PRIVATE_KEY],
      chainId: 204,
    }
  },
  etherscan: {
    apiKey: {
      bsc: process.env.BSCSCAN_API_KEY,
      bscTestnet: process.env.BSCSCAN_API_KEY,
    }
  }
};
```

### Добавление сети в MetaMask

```javascript
// BNB Chain Mainnet
async function addBNBChain() {
  await window.ethereum.request({
    method: 'wallet_addEthereumChain',
    params: [{
      chainId: '0x38', // 56
      chainName: 'BNB Smart Chain',
      nativeCurrency: {
        name: 'BNB',
        symbol: 'BNB',
        decimals: 18
      },
      rpcUrls: ['https://bsc-dataseed.binance.org'],
      blockExplorerUrls: ['https://bscscan.com']
    }]
  });
}

// opBNB (L2)
async function addOpBNB() {
  await window.ethereum.request({
    method: 'wallet_addEthereumChain',
    params: [{
      chainId: '0xCC', // 204
      chainName: 'opBNB Mainnet',
      nativeCurrency: {
        name: 'BNB',
        symbol: 'BNB',
        decimals: 18
      },
      rpcUrls: ['https://opbnb-mainnet-rpc.bnbchain.org'],
      blockExplorerUrls: ['https://opbnbscan.com']
    }]
  });
}
```

### BEP-20 Token Standard

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyBEP20Token
 * @notice BEP-20 полностью совместим с ERC-20
 * Разница только в сети деплоя
 */
contract MyBEP20Token is ERC20, Ownable {
    constructor() ERC20("My Token", "MTK") Ownable(msg.sender) {
        // Mint 1 миллион токенов
        _mint(msg.sender, 1_000_000 * 10**decimals());
    }

    // Дополнительная функция для BEP-20 (опционально)
    function getOwner() external view returns (address) {
        return owner();
    }

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }

    function burn(uint256 amount) external {
        _burn(msg.sender, amount);
    }
}
```

### Взаимодействие с контрактами

```javascript
// Пример swap на PancakeSwap
const { ethers } = require("ethers");

const PANCAKE_ROUTER = "0x10ED43C718714eb63d5aA57B78B54704E256024E"; // v2
const WBNB = "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c";
const BUSD = "0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56";

const routerABI = [
  "function swapExactETHForTokens(uint amountOutMin, address[] path, address to, uint deadline) payable returns (uint[] amounts)",
  "function getAmountsOut(uint amountIn, address[] path) view returns (uint[] amounts)"
];

async function swapBNBForBUSD(amountInBNB) {
  const provider = new ethers.JsonRpcProvider("https://bsc-dataseed.binance.org");
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  const router = new ethers.Contract(PANCAKE_ROUTER, routerABI, wallet);

  const amountIn = ethers.parseEther(amountInBNB);
  const path = [WBNB, BUSD];

  // Получаем ожидаемый output
  const amounts = await router.getAmountsOut(amountIn, path);
  const amountOutMin = amounts[1] * 95n / 100n; // 5% slippage

  const deadline = Math.floor(Date.now() / 1000) + 60 * 20; // 20 минут

  const tx = await router.swapExactETHForTokens(
    amountOutMin,
    path,
    wallet.address,
    deadline,
    { value: amountIn, gasPrice: ethers.parseUnits("3", "gwei") }
  );

  console.log("Swap tx:", tx.hash);
  await tx.wait();
  console.log("Swap completed!");
}
```

## opBNB - Layer 2

### Архитектура opBNB

```
┌─────────────────────────────────────────────────────────────┐
│                      opBNB Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  opBNB = OP Stack (Optimism) + BNB Chain                    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    opBNB (L2)                        │    │
│  │                                                      │    │
│  │  • TPS: 4000+                                       │    │
│  │  • Gas: <$0.001 per tx                              │    │
│  │  • Block time: 1 second                             │    │
│  │  • Optimized for gaming                             │    │
│  │                                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                    Data posting                              │
│                    & fraud proofs                            │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              BNB Smart Chain (L1)                    │    │
│  │                  Settlement                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Сравнение L1 vs L2

| Параметр | BNB Chain (L1) | opBNB (L2) |
|----------|----------------|------------|
| TPS | ~160 | ~4000 |
| Gas cost | $0.05-0.50 | $0.0001-0.01 |
| Block time | 3 сек | 1 сек |
| Finality | ~45 сек | ~7 дней* |
| Use case | General DeFi | Gaming, micro-txs |

*Финальность на L1, но транзакции подтверждаются мгновенно на L2

## Безопасность BNB Chain

### Известные уязвимости и инциденты

```
┌─────────────────────────────────────────────────────────────┐
│                  Security Considerations                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ⚠️ Централизация                                           │
│  • Только 21 активный валидатор                            │
│  • Binance контролирует значительную часть                 │
│                                                              │
│  ⚠️ Известные инциденты                                     │
│  • Октябрь 2022: Cross-chain bridge hack ($570M)           │
│  • Многочисленные rug pulls в DeFi                         │
│                                                              │
│  ✅ Меры безопасности                                       │
│  • Аудиты от CertiK, PeckShield                            │
│  • Bug bounty программа                                     │
│  • Real-time monitoring                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Плюсы и минусы

### Преимущества BNB Chain

| Плюс | Описание |
|------|----------|
| **Низкие комиссии** | ~$0.05-0.50 за транзакцию |
| **Высокая скорость** | 3 сек block time |
| **EVM совместимость** | Легкая миграция из Ethereum |
| **Поддержка Binance** | Ресурсы крупнейшей биржи |
| **Большая экосистема** | Много DeFi протоколов |
| **opBNB** | L2 для высоконагруженных приложений |

### Недостатки

| Минус | Описание |
|-------|----------|
| **Централизация** | 21 валидатор, влияние Binance |
| **Регуляторные риски** | Проблемы Binance с регуляторами |
| **Безопасность** | Меньше децентрализации = больше рисков |
| **Качество проектов** | Много scam и rug pulls |

## Сравнение с Ethereum

| Характеристика | BNB Chain | Ethereum |
|----------------|-----------|----------|
| **Консенсус** | PoSA (21 val) | PoS (900k+ val) |
| **Децентрализация** | Низкая | Высокая |
| **TPS** | ~160 | ~15-30 |
| **Avg Gas Fee** | $0.05-0.50 | $1-50 |
| **Block Time** | 3 сек | 12 сек |
| **TVL** | ~$5B | ~$50B |
| **Экосистема** | Развитая | Крупнейшая |

## Полезные ресурсы

- [bnbchain.org](https://www.bnbchain.org) - официальный сайт
- [docs.bnbchain.org](https://docs.bnbchain.org) - документация
- [BscScan](https://bscscan.com) - блок-эксплорер
- [opBNBScan](https://opbnbscan.com) - эксплорер opBNB
- [Testnet Faucet](https://testnet.bnbchain.org/faucet-smart) - тестовые BNB
- [PancakeSwap](https://pancakeswap.finance) - главный DEX
