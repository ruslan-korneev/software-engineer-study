# Ethereum

## Введение

**Ethereum** - это децентрализованная платформа для создания смарт-контрактов и децентрализованных приложений (dApps). Ethereum был первым блокчейном, реализовавшим полноценную виртуальную машину (EVM), способную выполнять произвольный код.

## История создания

### Хронология

```
┌─────────────────────────────────────────────────────────────┐
│                 Ethereum Timeline                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  2013          2014           2015          2022            │
│   │             │              │             │               │
│   ▼             ▼              ▼             ▼               │
│ ┌────┐       ┌────┐        ┌────┐       ┌────┐              │
│ │White│       │Yellow│      │Genesis│    │The  │              │
│ │Paper│       │Paper │      │Block  │    │Merge│              │
│ └────┘       └────┘        └────┘       └────┘              │
│                                                              │
│ Виталик       Гэвин        Запуск       PoW → PoS           │
│ Бутерин       Вуд          mainnet                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Ключевые даты

| Дата | Событие |
|------|---------|
| Ноябрь 2013 | Виталик Бутерин публикует White Paper |
| Апрель 2014 | Гэвин Вуд публикует Yellow Paper (техническая спецификация) |
| Июль-Август 2014 | ICO - собрано ~$18 млн |
| 30 июля 2015 | Запуск mainnet (Frontier) |
| Июнь 2016 | DAO Hack и хардфорк (появление Ethereum Classic) |
| 15 сентября 2022 | The Merge - переход на Proof of Stake |
| Март 2024 | Dencun upgrade (Proto-Danksharding) |

### Основатели

- **Виталик Бутерин** - автор идеи и White Paper
- **Гэвин Вуд** - автор Yellow Paper и Solidity
- **Чарльз Хоскинсон** - позже создал Cardano
- **Джозеф Любин** - основатель ConsenSys

## Технические характеристики

### Консенсус: Proof of Stake (после The Merge)

```
┌─────────────────────────────────────────────────────────────┐
│               Ethereum Proof of Stake                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Минимальный стейк: 32 ETH                                  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Validator Pool                       │   │
│  │                                                       │   │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐        │   │
│  │  │ 32  │  │ 32  │  │ 32  │  │ 32  │  │ 32  │        │   │
│  │  │ ETH │  │ ETH │  │ ETH │  │ ETH │  │ ETH │        │   │
│  │  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘        │   │
│  │     │         │         │         │         │        │   │
│  │     └─────────┴────┬────┴─────────┘         │        │   │
│  │                    │                                 │   │
│  │               Proposer                               │   │
│  │                  │                                   │   │
│  │                  ▼                                   │   │
│  │            ┌──────────┐                              │   │
│  │            │  Block   │                              │   │
│  │            │ Proposal │                              │   │
│  │            └──────────┘                              │   │
│  │                  │                                   │   │
│  │                  ▼                                   │   │
│  │         Attestation by                               │   │
│  │         Committee                                    │   │
│  │                                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Основные параметры сети

| Параметр | Значение |
|----------|----------|
| Block Time | ~12 секунд |
| Finality | ~13 минут (2 epochs) |
| TPS (базовый) | ~15-30 |
| TPS (с L2) | ~2000-7000 |
| Gas Limit | ~30M gas/block |
| Минимальный стейк | 32 ETH |

### Система Gas

Gas - это единица измерения вычислительной работы в Ethereum.

```
┌─────────────────────────────────────────────────────────────┐
│                    Gas Price Formula                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Transaction Fee = Gas Used × Gas Price                    │
│                                                              │
│   Post EIP-1559:                                            │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Transaction Fee = Gas Used × (Base Fee + Tip)       │   │
│   │                                                      │   │
│   │ • Base Fee - сжигается (burned)                     │   │
│   │ • Tip (Priority Fee) - идет валидатору              │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```solidity
// Примеры стоимости операций в gas
// SSTORE (запись в storage): 20,000 gas (новое значение)
// SSTORE (обновление): 5,000 gas
// SLOAD (чтение): 2,100 gas
// CALL (вызов контракта): 2,600 gas + исполнение
// ETH transfer: 21,000 gas

// Пример оптимизации gas
contract GasOptimization {
    // Плохо: много записей в storage
    uint256 public counter;
    function badIncrement() external {
        for (uint i = 0; i < 10; i++) {
            counter++; // 10 SSTORE операций!
        }
    }

    // Хорошо: одна запись в storage
    function goodIncrement() external {
        uint256 temp = counter;
        for (uint i = 0; i < 10; i++) {
            temp++;
        }
        counter = temp; // 1 SSTORE операция
    }
}
```

## Смарт-контракты

### Что такое смарт-контракт?

Смарт-контракт - это программа, хранящаяся в блокчейне, которая автоматически исполняется при выполнении заданных условий.

```
┌─────────────────────────────────────────────────────────────┐
│                Smart Contract Lifecycle                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│   │ Solidity │ →  │ Compiler │ →  │ Bytecode │              │
│   │   Code   │    │ (solc)   │    │   +ABI   │              │
│   └──────────┘    └──────────┘    └──────────┘              │
│                                         │                    │
│                                         ▼                    │
│                                  ┌──────────┐               │
│                                  │  Deploy  │               │
│                                  │Transaction│               │
│                                  └──────────┘               │
│                                         │                    │
│                                         ▼                    │
│                                  ┌──────────┐               │
│                                  │ Contract │               │
│                                  │ Address  │               │
│                                  │0x1234... │               │
│                                  └──────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Пример простого контракта

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title SimpleStorage
 * @notice Простой контракт для хранения и чтения числа
 */
contract SimpleStorage {
    // Состояние контракта (storage)
    uint256 private storedValue;
    address public owner;

    // События для логирования
    event ValueChanged(uint256 oldValue, uint256 newValue, address changedBy);

    // Модификатор доступа
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }

    // Конструктор - вызывается один раз при деплое
    constructor(uint256 initialValue) {
        storedValue = initialValue;
        owner = msg.sender;
    }

    // Функция записи (изменяет state, требует gas)
    function setValue(uint256 newValue) external onlyOwner {
        uint256 oldValue = storedValue;
        storedValue = newValue;
        emit ValueChanged(oldValue, newValue, msg.sender);
    }

    // Функция чтения (view - не требует gas при внешнем вызове)
    function getValue() external view returns (uint256) {
        return storedValue;
    }
}
```

### Стандарты токенов

```solidity
// ERC-20: Fungible Tokens (взаимозаменяемые)
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// ERC-721: Non-Fungible Tokens (NFT)
interface IERC721 {
    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address);

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
}

// ERC-1155: Multi-Token Standard (fungible + non-fungible в одном)
interface IERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids)
        external view returns (uint256[] memory);
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data)
        external;
}
```

## Экосистема Ethereum

### DeFi (Decentralized Finance)

```
┌─────────────────────────────────────────────────────────────┐
│                  DeFi Ecosystem                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │     DEXes      │  │    Lending     │  │   Stablecoins  │ │
│  │                │  │                │  │                │ │
│  │ • Uniswap      │  │ • Aave         │  │ • USDC         │ │
│  │ • Curve        │  │ • Compound     │  │ • DAI          │ │
│  │ • Balancer     │  │ • MakerDAO     │  │ • USDT         │ │
│  │ • SushiSwap    │  │ • Morpho       │  │ • FRAX         │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │   Derivatives  │  │  Aggregators   │  │    Bridges     │ │
│  │                │  │                │  │                │ │
│  │ • GMX          │  │ • 1inch        │  │ • Stargate     │ │
│  │ • dYdX         │  │ • Paraswap     │  │ • Across       │ │
│  │ • Synthetix    │  │ • CoW Protocol │  │ • Hop          │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### NFT и Gaming

- **Маркетплейсы**: OpenSea, Blur, LooksRare
- **Коллекции**: CryptoPunks, Bored Ape Yacht Club, Azuki
- **Gaming**: Axie Infinity, Gods Unchained, Illuvium

### DAOs (Decentralized Autonomous Organizations)

- **Governance**: Compound, Uniswap, Aave DAO
- **Investment**: The LAO, MetaCartel
- **Social**: ENS DAO, Gitcoin

## Инструменты разработки

### Фреймворки

```bash
# Hardhat - наиболее популярный
npm init -y
npm install --save-dev hardhat
npx hardhat init

# Foundry - быстрый, написан на Rust
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge init my-project
```

### Структура Hardhat проекта

```
my-project/
├── contracts/           # Solidity контракты
│   └── MyContract.sol
├── scripts/             # Скрипты деплоя
│   └── deploy.js
├── test/                # Тесты
│   └── MyContract.test.js
├── hardhat.config.js    # Конфигурация
└── package.json
```

### hardhat.config.js

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    hardhat: {
      // Локальная сеть для тестов
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    },
    mainnet: {
      url: process.env.MAINNET_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    }
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  }
};
```

### Скрипт деплоя

```javascript
const { ethers } = require("hardhat");

async function main() {
  console.log("Deploying SimpleStorage...");

  const SimpleStorage = await ethers.getContractFactory("SimpleStorage");
  const simpleStorage = await SimpleStorage.deploy(42); // initialValue = 42

  await simpleStorage.waitForDeployment();

  const address = await simpleStorage.getAddress();
  console.log("SimpleStorage deployed to:", address);

  // Верификация на Etherscan
  if (network.name !== "hardhat") {
    console.log("Waiting for block confirmations...");
    await simpleStorage.deploymentTransaction().wait(6);

    await run("verify:verify", {
      address: address,
      constructorArguments: [42]
    });
  }
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

### Тестирование

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleStorage", function () {
  let simpleStorage;
  let owner;
  let otherAccount;

  beforeEach(async function () {
    [owner, otherAccount] = await ethers.getSigners();

    const SimpleStorage = await ethers.getContractFactory("SimpleStorage");
    simpleStorage = await SimpleStorage.deploy(42);
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await simpleStorage.owner()).to.equal(owner.address);
    });

    it("Should set the initial value", async function () {
      expect(await simpleStorage.getValue()).to.equal(42);
    });
  });

  describe("setValue", function () {
    it("Should update the value when called by owner", async function () {
      await simpleStorage.setValue(100);
      expect(await simpleStorage.getValue()).to.equal(100);
    });

    it("Should revert when called by non-owner", async function () {
      await expect(
        simpleStorage.connect(otherAccount).setValue(100)
      ).to.be.revertedWith("Not the owner");
    });

    it("Should emit ValueChanged event", async function () {
      await expect(simpleStorage.setValue(100))
        .to.emit(simpleStorage, "ValueChanged")
        .withArgs(42, 100, owner.address);
    });
  });
});
```

## Foundry (альтернатива Hardhat)

```bash
# Создание проекта
forge init my-foundry-project
cd my-foundry-project

# Структура
# ├── src/           # Контракты
# ├── test/          # Тесты (на Solidity!)
# ├── script/        # Скрипты
# └── foundry.toml   # Конфигурация
```

```solidity
// test/SimpleStorage.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/SimpleStorage.sol";

contract SimpleStorageTest is Test {
    SimpleStorage public simpleStorage;
    address public owner = address(this);
    address public user = address(0x1);

    function setUp() public {
        simpleStorage = new SimpleStorage(42);
    }

    function test_InitialValue() public view {
        assertEq(simpleStorage.getValue(), 42);
    }

    function test_SetValue() public {
        simpleStorage.setValue(100);
        assertEq(simpleStorage.getValue(), 100);
    }

    function testFail_SetValueNotOwner() public {
        vm.prank(user);
        simpleStorage.setValue(100);
    }

    // Fuzz testing!
    function testFuzz_SetValue(uint256 newValue) public {
        simpleStorage.setValue(newValue);
        assertEq(simpleStorage.getValue(), newValue);
    }
}
```

```bash
# Команды Foundry
forge build          # Компиляция
forge test           # Тесты
forge test -vvv      # Verbose output
forge test --gas-report  # Gas отчет

# Деплой
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast

# Верификация
forge verify-contract $ADDRESS SimpleStorage --etherscan-api-key $KEY
```

## Плюсы и минусы Ethereum

### Преимущества

| Плюс | Описание |
|------|----------|
| **Децентрализация** | Тысячи нод по всему миру |
| **Безопасность** | Проверенная временем сеть с 2015 года |
| **Экосистема** | Крупнейшая экосистема DeFi, NFT, DAO |
| **Ликвидность** | Наибольший TVL среди всех сетей |
| **Разработчики** | Самое большое сообщество |
| **L2 решения** | Rollups для масштабирования |

### Недостатки

| Минус | Описание |
|-------|----------|
| **Высокие комиссии** | $1-50+ за транзакцию в пиковые моменты |
| **Медленные транзакции** | ~12 сек на блок, финальность ~13 мин |
| **Сложность** | Высокий порог входа для разработчиков |
| **Scalability** | Базовый TPS низкий (15-30) |

## Layer 2 решения

```
┌─────────────────────────────────────────────────────────────┐
│                   Ethereum L2 Landscape                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌────────────────────────────────────────────────────┐    │
│   │              Ethereum (Layer 1)                     │    │
│   │                Settlement Layer                     │    │
│   └────────────────────────────────────────────────────┘    │
│                          ▲                                   │
│            ┌─────────────┴─────────────┐                    │
│            │                           │                    │
│   ┌────────────────┐         ┌────────────────┐            │
│   │ Optimistic     │         │  ZK Rollups    │            │
│   │ Rollups        │         │                │            │
│   │                │         │                │            │
│   │ • Arbitrum     │         │ • zkSync Era   │            │
│   │ • Optimism     │         │ • StarkNet     │            │
│   │ • Base         │         │ • Polygon zkEVM│            │
│   │ • Mantle       │         │ • Linea        │            │
│   └────────────────┘         │ • Scroll       │            │
│                              └────────────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Полезные ресурсы

- [ethereum.org](https://ethereum.org) - официальный сайт
- [Solidity Docs](https://docs.soliditylang.org) - документация Solidity
- [Etherscan](https://etherscan.io) - блок-эксплорер
- [Hardhat](https://hardhat.org) - фреймворк разработки
- [Foundry](https://book.getfoundry.sh) - альтернативный фреймворк
- [OpenZeppelin](https://docs.openzeppelin.com) - библиотека контрактов
