# Hardhat

## Введение

**Hardhat** - это современная среда разработки для Ethereum, которая позволяет компилировать, деплоить, тестировать и отлаживать смарт-контракты. Hardhat является одним из самых популярных инструментов в экосистеме Ethereum благодаря своей гибкости, расширяемости и отличному developer experience.

## Преимущества Hardhat

1. **Быстрая компиляция** - инкрементальная компиляция и кэширование
2. **Отладка Solidity** - stack traces и console.log прямо в контрактах
3. **Гибкая система плагинов** - расширение функциональности
4. **TypeScript поддержка** - нативная поддержка из коробки
5. **Hardhat Network** - встроенная локальная сеть с возможностью форка mainnet
6. **Активное сообщество** - большое количество плагинов и примеров

## Установка и инициализация проекта

### Требования

- Node.js >= 16.0
- npm или yarn

### Создание проекта

```bash
# Создание директории проекта
mkdir my-hardhat-project
cd my-hardhat-project

# Инициализация npm проекта
npm init -y

# Установка Hardhat
npm install --save-dev hardhat

# Инициализация Hardhat проекта
npx hardhat init
```

При инициализации Hardhat предложит несколько опций:

```
? What do you want to do? ...
> Create a JavaScript project
  Create a TypeScript project
  Create a TypeScript project (with Viem)
  Create an empty hardhat.config.js
  Quit
```

Рекомендуется выбирать TypeScript проект для лучшей типизации.

## Структура проекта

```
my-hardhat-project/
├── contracts/           # Solidity контракты
│   └── Lock.sol
├── scripts/             # Скрипты деплоя и взаимодействия
│   └── deploy.ts
├── test/                # Тесты
│   └── Lock.ts
├── artifacts/           # Скомпилированные контракты (генерируется)
├── cache/               # Кэш компиляции (генерируется)
├── typechain-types/     # TypeScript типы (генерируется)
├── hardhat.config.ts    # Конфигурация Hardhat
├── package.json
└── tsconfig.json
```

## Конфигурация hardhat.config.ts

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const config: HardhatUserConfig = {
  // Версия компилятора Solidity
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
      viaIR: true, // Использовать IR pipeline
    },
  },

  // Настройки сетей
  networks: {
    // Локальная сеть (по умолчанию)
    hardhat: {
      chainId: 31337,
      // Форк mainnet
      forking: {
        url: process.env.MAINNET_RPC_URL || "",
        blockNumber: 18500000, // Опционально: фиксированный блок
        enabled: false, // Включить при необходимости
      },
    },
    // Localhost (для отдельной ноды)
    localhost: {
      url: "http://127.0.0.1:8545",
    },
    // Sepolia testnet
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 11155111,
    },
    // Ethereum mainnet
    mainnet: {
      url: process.env.MAINNET_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 1,
    },
  },

  // Настройки Etherscan для верификации
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY || "",
      sepolia: process.env.ETHERSCAN_API_KEY || "",
    },
  },

  // Пути к директориям
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts",
  },

  // Gas Reporter
  gasReporter: {
    enabled: true,
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
  },
};

export default config;
```

## Компиляция контрактов

```bash
# Компиляция всех контрактов
npx hardhat compile

# Принудительная перекомпиляция
npx hardhat compile --force

# Очистка артефактов
npx hardhat clean
```

### Пример контракта

```solidity
// contracts/Token.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    constructor(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) ERC20(name, symbol) Ownable(msg.sender) {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```

## Тестирование (Mocha + Chai)

Hardhat использует Mocha как test runner и Chai для assertions.

### Базовый тест

```typescript
// test/Token.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { MyToken } from "../typechain-types";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";

describe("MyToken", function () {
  let token: MyToken;
  let owner: SignerWithAddress;
  let addr1: SignerWithAddress;
  let addr2: SignerWithAddress;

  beforeEach(async function () {
    // Получение аккаунтов
    [owner, addr1, addr2] = await ethers.getSigners();

    // Деплой контракта
    const Token = await ethers.getContractFactory("MyToken");
    token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.waitForDeployment();
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await token.owner()).to.equal(owner.address);
    });

    it("Should assign the total supply to the owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });
  });

  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      // Transfer 50 tokens from owner to addr1
      await token.transfer(addr1.address, 50);
      expect(await token.balanceOf(addr1.address)).to.equal(50);

      // Transfer 50 tokens from addr1 to addr2
      await token.connect(addr1).transfer(addr2.address, 50);
      expect(await token.balanceOf(addr2.address)).to.equal(50);
    });

    it("Should fail if sender doesn't have enough tokens", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);

      // Try to send 1 token from addr1 (0 tokens) to owner
      await expect(
        token.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWithCustomError(token, "ERC20InsufficientBalance");

      // Owner balance shouldn't have changed
      expect(await token.balanceOf(owner.address)).to.equal(initialOwnerBalance);
    });
  });

  describe("Minting", function () {
    it("Should mint tokens by owner", async function () {
      await token.mint(addr1.address, 1000);
      expect(await token.balanceOf(addr1.address)).to.equal(1000);
    });

    it("Should fail if non-owner tries to mint", async function () {
      await expect(
        token.connect(addr1).mint(addr1.address, 1000)
      ).to.be.revertedWithCustomError(token, "OwnableUnauthorizedAccount");
    });
  });
});
```

### Тестирование с time manipulation

```typescript
import { time } from "@nomicfoundation/hardhat-network-helpers";

describe("TimeLock", function () {
  it("Should unlock after time passes", async function () {
    const unlockTime = (await time.latest()) + 60 * 60; // 1 hour

    const Lock = await ethers.getContractFactory("Lock");
    const lock = await Lock.deploy(unlockTime, { value: 1000 });

    // Перемотка времени на 1 час
    await time.increase(60 * 60);

    await expect(lock.withdraw()).to.not.be.reverted;
  });
});
```

### Запуск тестов

```bash
# Запуск всех тестов
npx hardhat test

# Запуск конкретного теста
npx hardhat test test/Token.test.ts

# Запуск с gas reporting
REPORT_GAS=true npx hardhat test

# Запуск с coverage
npx hardhat coverage
```

## Деплой скрипты

```typescript
// scripts/deploy.ts
import { ethers, run, network } from "hardhat";

async function main() {
  console.log("Deploying MyToken...");

  const [deployer] = await ethers.getSigners();
  console.log("Deploying with account:", deployer.address);

  const balance = await ethers.provider.getBalance(deployer.address);
  console.log("Account balance:", ethers.formatEther(balance), "ETH");

  // Деплой контракта
  const Token = await ethers.getContractFactory("MyToken");
  const token = await Token.deploy("MyToken", "MTK", 1000000);
  await token.waitForDeployment();

  const tokenAddress = await token.getAddress();
  console.log("MyToken deployed to:", tokenAddress);

  // Верификация на Etherscan (только для публичных сетей)
  if (network.name !== "hardhat" && network.name !== "localhost") {
    console.log("Waiting for block confirmations...");
    await token.deploymentTransaction()?.wait(6);

    console.log("Verifying contract on Etherscan...");
    try {
      await run("verify:verify", {
        address: tokenAddress,
        constructorArguments: ["MyToken", "MTK", 1000000],
      });
      console.log("Contract verified!");
    } catch (error: any) {
      if (error.message.includes("Already Verified")) {
        console.log("Contract already verified");
      } else {
        console.error("Verification error:", error);
      }
    }
  }
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### Запуск деплоя

```bash
# Деплой на локальную сеть
npx hardhat run scripts/deploy.ts

# Деплой на Sepolia
npx hardhat run scripts/deploy.ts --network sepolia

# Деплой на mainnet
npx hardhat run scripts/deploy.ts --network mainnet
```

## Hardhat Network

Hardhat Network - это встроенная локальная Ethereum сеть для разработки.

### Запуск локальной ноды

```bash
# Запуск ноды
npx hardhat node

# Запуск с форком mainnet
npx hardhat node --fork https://mainnet.infura.io/v3/YOUR_KEY
```

### Форк Mainnet

Форк позволяет тестировать взаимодействие с существующими контрактами:

```typescript
// hardhat.config.ts
hardhat: {
  forking: {
    url: "https://mainnet.infura.io/v3/YOUR_KEY",
    blockNumber: 18500000, // Фиксированный блок для детерминизма
  },
}
```

### Пример теста с форком

```typescript
describe("Uniswap Integration", function () {
  it("Should swap ETH for DAI on Uniswap", async function () {
    const UNISWAP_ROUTER = "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D";
    const DAI = "0x6B175474E89094C44Da98b954EescdeCB5BE3F3E";
    const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";

    const router = await ethers.getContractAt("IUniswapV2Router02", UNISWAP_ROUTER);

    const [signer] = await ethers.getSigners();
    const deadline = Math.floor(Date.now() / 1000) + 60 * 20;

    await router.swapExactETHForTokens(
      0, // min amount out
      [WETH, DAI],
      signer.address,
      deadline,
      { value: ethers.parseEther("1") }
    );
  });
});
```

## Основные плагины

### @nomicfoundation/hardhat-toolbox

Мета-плагин, включающий основные инструменты:

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox
```

Включает:
- hardhat-ethers
- hardhat-chai-matchers
- hardhat-network-helpers
- hardhat-verify
- hardhat-gas-reporter
- solidity-coverage
- typechain

### hardhat-gas-reporter

Отчет о расходе газа:

```typescript
// hardhat.config.ts
gasReporter: {
  enabled: process.env.REPORT_GAS === "true",
  currency: "USD",
  coinmarketcap: process.env.COINMARKETCAP_API_KEY,
  outputFile: "gas-report.txt",
  noColors: true,
}
```

### hardhat-deploy

Продвинутая система деплоя:

```bash
npm install --save-dev hardhat-deploy
```

```typescript
// deploy/001_deploy_token.ts
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { DeployFunction } from "hardhat-deploy/types";

const func: DeployFunction = async function (hre: HardhatRuntimeEnvironment) {
  const { deployments, getNamedAccounts } = hre;
  const { deploy } = deployments;
  const { deployer } = await getNamedAccounts();

  await deploy("MyToken", {
    from: deployer,
    args: ["MyToken", "MTK", 1000000],
    log: true,
    autoMine: true,
  });
};

export default func;
func.tags = ["MyToken"];
```

## Console.log в Solidity

Hardhat позволяет использовать console.log прямо в Solidity контрактах:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "hardhat/console.sol";

contract MyContract {
    function doSomething(uint256 value) public {
        console.log("Function called with value:", value);
        console.log("Sender:", msg.sender);
        console.log("Block number:", block.number);

        // Поддерживаются разные типы
        console.logBytes32(keccak256(abi.encodePacked(value)));
        console.logBool(value > 100);
    }
}
```

**Важно:** Убедитесь, что `console.log` удален перед деплоем на mainnet!

## Полезные команды Hardhat

```bash
# Список доступных задач
npx hardhat help

# Информация о конкретной задаче
npx hardhat help compile

# Консоль для взаимодействия с контрактами
npx hardhat console --network localhost

# Получение размера контрактов
npx hardhat size-contracts

# Очистка кэша и артефактов
npx hardhat clean
```

## Лучшие практики

1. **Используйте TypeScript** - лучшая типизация и автодополнение
2. **Настройте .env** - не храните приватные ключи в коде
3. **Используйте fixtures** - для переиспользования setup в тестах
4. **Пишите comprehensive тесты** - покрывайте edge cases
5. **Используйте hardhat-deploy** - для сложных систем деплоя
6. **Настройте CI/CD** - автоматические тесты и деплой

## Ресурсы

- [Официальная документация](https://hardhat.org/docs)
- [Hardhat GitHub](https://github.com/NomicFoundation/hardhat)
- [Hardhat Tutorials](https://hardhat.org/tutorial)
- [Плагины Hardhat](https://hardhat.org/hardhat-runner/plugins)
