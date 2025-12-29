# Truffle

## Введение

**Truffle** - это один из первых и исторически важных фреймворков для разработки смарт-контрактов на Ethereum. Truffle Suite включает в себя набор инструментов для компиляции, тестирования, деплоя и управления смарт-контрактами.

> **Важно:** В 2023 году компания Consensys (разработчик Truffle) объявила о прекращении активной разработки Truffle Suite. Рекомендуется использовать Hardhat или Foundry для новых проектов. Тем не менее, понимание Truffle полезно для работы с существующими проектами.

## Компоненты Truffle Suite

1. **Truffle** - основной фреймворк для разработки
2. **Ganache** - локальная блокчейн сеть для разработки
3. **Drizzle** - библиотека для фронтенд интеграции (deprecated)

## Установка и создание проекта

### Требования

- Node.js >= 14.0
- npm или yarn

### Установка Truffle

```bash
# Глобальная установка
npm install -g truffle

# Проверка версии
truffle version
```

### Создание проекта

```bash
# Создание пустого проекта
mkdir my-truffle-project
cd my-truffle-project
truffle init

# Создание из шаблона (Truffle Box)
truffle unbox metacoin

# Популярные boxes
truffle unbox react     # React frontend
truffle unbox pet-shop  # Tutorial проект
```

## Структура проекта

```
my-truffle-project/
├── contracts/              # Solidity контракты
│   └── Migrations.sol      # Служебный контракт для миграций
├── migrations/             # Скрипты деплоя (миграции)
│   └── 1_deploy_contracts.js
├── test/                   # Тесты (JS или Solidity)
│   └── test_example.js
├── build/                  # Скомпилированные контракты (генерируется)
│   └── contracts/
├── truffle-config.js       # Конфигурация проекта
└── package.json
```

### Контракт Migrations

```solidity
// contracts/Migrations.sol
// SPDX-License-Identifier: MIT
pragma solidity >=0.4.22 <0.9.0;

contract Migrations {
    address public owner = msg.sender;
    uint public last_completed_migration;

    modifier restricted() {
        require(
            msg.sender == owner,
            "This function is restricted to the contract's owner"
        );
        _;
    }

    function setCompleted(uint completed) public restricted {
        last_completed_migration = completed;
    }
}
```

## Конфигурация truffle-config.js

```javascript
// truffle-config.js
require('dotenv').config();
const HDWalletProvider = require('@truffle/hdwallet-provider');

module.exports = {
  // Настройки сетей
  networks: {
    // Локальная разработка
    development: {
      host: "127.0.0.1",
      port: 8545,        // Стандартный порт Ganache
      network_id: "*",   // Любой network id
    },

    // Ganache GUI
    ganache: {
      host: "127.0.0.1",
      port: 7545,        // Порт Ganache GUI
      network_id: 5777,
    },

    // Sepolia testnet
    sepolia: {
      provider: () => new HDWalletProvider(
        process.env.MNEMONIC,
        process.env.SEPOLIA_RPC_URL
      ),
      network_id: 11155111,
      gas: 5500000,
      confirmations: 2,
      timeoutBlocks: 200,
      skipDryRun: true,
    },

    // Ethereum mainnet
    mainnet: {
      provider: () => new HDWalletProvider(
        process.env.MNEMONIC,
        process.env.MAINNET_RPC_URL
      ),
      network_id: 1,
      gas: 5500000,
      gasPrice: 20000000000, // 20 gwei
      confirmations: 2,
      timeoutBlocks: 200,
      skipDryRun: false,
    },
  },

  // Настройки компилятора
  compilers: {
    solc: {
      version: "0.8.24",
      settings: {
        optimizer: {
          enabled: true,
          runs: 200,
        },
        evmVersion: "paris",
      },
    },
  },

  // Плагины
  plugins: [
    'truffle-plugin-verify',
    'truffle-contract-size',
  ],

  // API ключи для верификации
  api_keys: {
    etherscan: process.env.ETHERSCAN_API_KEY,
  },

  // Пути к директориям
  contracts_directory: './contracts',
  contracts_build_directory: './build/contracts',
  migrations_directory: './migrations',
  test_directory: './test',
};
```

## Миграции (Deployments)

Миграции - это скрипты для деплоя контрактов. Они выполняются последовательно и отслеживают, какие миграции уже выполнены.

### Базовая миграция

```javascript
// migrations/1_deploy_contracts.js
const Migrations = artifacts.require("Migrations");

module.exports = function(deployer) {
  deployer.deploy(Migrations);
};
```

### Деплой с параметрами

```javascript
// migrations/2_deploy_token.js
const MyToken = artifacts.require("MyToken");

module.exports = async function(deployer, network, accounts) {
  const name = "MyToken";
  const symbol = "MTK";
  const initialSupply = web3.utils.toWei("1000000", "ether");

  await deployer.deploy(MyToken, name, symbol, initialSupply);

  const token = await MyToken.deployed();
  console.log("Token deployed at:", token.address);
};
```

### Деплой связанных контрактов

```javascript
// migrations/3_deploy_dex.js
const Token = artifacts.require("Token");
const DEX = artifacts.require("DEX");

module.exports = async function(deployer, network, accounts) {
  // Деплой токена
  await deployer.deploy(Token, "MyToken", "MTK", "1000000000000000000000000");
  const token = await Token.deployed();

  // Деплой DEX с адресом токена
  await deployer.deploy(DEX, token.address);
  const dex = await DEX.deployed();

  // Настройка: разрешить DEX тратить токены
  await token.approve(dex.address, "1000000000000000000000000");

  console.log("Token:", token.address);
  console.log("DEX:", dex.address);
};
```

### Условный деплой

```javascript
// migrations/4_conditional_deploy.js
const Oracle = artifacts.require("Oracle");
const MockOracle = artifacts.require("MockOracle");

module.exports = async function(deployer, network) {
  if (network === "development" || network === "test") {
    // Деплой mock для тестов
    await deployer.deploy(MockOracle);
  } else {
    // Использование реального оракула на mainnet
    await deployer.deploy(Oracle, "0xRealOracleAddress");
  }
};
```

### Запуск миграций

```bash
# Запуск всех миграций
truffle migrate

# На конкретной сети
truffle migrate --network sepolia

# Перезапуск всех миграций
truffle migrate --reset

# Запуск с определенной миграции
truffle migrate --f 2 --to 3
```

## Тестирование

Truffle поддерживает тесты на JavaScript (Mocha) и Solidity.

### JavaScript тесты

```javascript
// test/Token.test.js
const Token = artifacts.require("Token");
const truffleAssert = require('truffle-assertions');

contract("Token", (accounts) => {
  const [owner, user1, user2] = accounts;
  let token;

  beforeEach(async () => {
    token = await Token.new("MyToken", "MTK", web3.utils.toWei("1000000", "ether"));
  });

  describe("Deployment", () => {
    it("should set the correct name and symbol", async () => {
      const name = await token.name();
      const symbol = await token.symbol();

      assert.equal(name, "MyToken");
      assert.equal(symbol, "MTK");
    });

    it("should assign total supply to owner", async () => {
      const totalSupply = await token.totalSupply();
      const ownerBalance = await token.balanceOf(owner);

      assert.equal(ownerBalance.toString(), totalSupply.toString());
    });
  });

  describe("Transfers", () => {
    it("should transfer tokens between accounts", async () => {
      const amount = web3.utils.toWei("100", "ether");

      await token.transfer(user1, amount, { from: owner });

      const user1Balance = await token.balanceOf(user1);
      assert.equal(user1Balance.toString(), amount);
    });

    it("should fail when sender doesn't have enough balance", async () => {
      const amount = web3.utils.toWei("100", "ether");

      await truffleAssert.reverts(
        token.transfer(user2, amount, { from: user1 }),
        "ERC20: insufficient balance"
      );
    });

    it("should emit Transfer event", async () => {
      const amount = web3.utils.toWei("100", "ether");

      const result = await token.transfer(user1, amount, { from: owner });

      truffleAssert.eventEmitted(result, 'Transfer', (ev) => {
        return ev.from === owner &&
               ev.to === user1 &&
               ev.value.toString() === amount;
      });
    });
  });

  describe("Allowances", () => {
    it("should approve and transferFrom", async () => {
      const amount = web3.utils.toWei("100", "ether");

      await token.approve(user1, amount, { from: owner });
      await token.transferFrom(owner, user2, amount, { from: user1 });

      const user2Balance = await token.balanceOf(user2);
      assert.equal(user2Balance.toString(), amount);
    });
  });
});
```

### Тесты с временем и блоками

```javascript
// test/TimeLock.test.js
const TimeLock = artifacts.require("TimeLock");
const { time } = require('@openzeppelin/test-helpers');

contract("TimeLock", (accounts) => {
  const [owner] = accounts;
  let lock;
  let unlockTime;

  beforeEach(async () => {
    const latestTime = await time.latest();
    unlockTime = latestTime.add(time.duration.hours(1));

    lock = await TimeLock.new(unlockTime, {
      from: owner,
      value: web3.utils.toWei("1", "ether")
    });
  });

  it("should not allow withdrawal before unlock time", async () => {
    await truffleAssert.reverts(
      lock.withdraw({ from: owner }),
      "Too early"
    );
  });

  it("should allow withdrawal after unlock time", async () => {
    // Перемотка времени
    await time.increaseTo(unlockTime.add(time.duration.seconds(1)));

    const balanceBefore = await web3.eth.getBalance(owner);
    await lock.withdraw({ from: owner });
    const balanceAfter = await web3.eth.getBalance(owner);

    assert(BigInt(balanceAfter) > BigInt(balanceBefore));
  });
});
```

### Solidity тесты

```solidity
// test/TestToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "truffle/Assert.sol";
import "truffle/DeployedAddresses.sol";
import "../contracts/Token.sol";

contract TestToken {
    Token token;

    function beforeEach() public {
        token = new Token("Test", "TST", 1000000 * 10**18);
    }

    function testInitialSupply() public {
        uint256 expected = 1000000 * 10**18;
        Assert.equal(token.totalSupply(), expected, "Initial supply should be 1M");
    }

    function testTransfer() public {
        address recipient = address(0x1);
        uint256 amount = 100 * 10**18;

        token.transfer(recipient, amount);

        Assert.equal(token.balanceOf(recipient), amount, "Recipient should have 100 tokens");
    }
}
```

### Запуск тестов

```bash
# Запуск всех тестов
truffle test

# Запуск конкретного файла
truffle test test/Token.test.js

# На конкретной сети
truffle test --network development

# С отладкой
truffle test --debug
```

## Ganache

Ganache - локальная блокчейн сеть для разработки.

### Ganache CLI

```bash
# Установка
npm install -g ganache

# Запуск
ganache

# С параметрами
ganache \
  --port 8545 \
  --accounts 10 \
  --defaultBalanceEther 1000 \
  --gasLimit 8000000 \
  --blockTime 0

# С форком mainnet
ganache --fork https://mainnet.infura.io/v3/YOUR_KEY

# С детерминированными аккаунтами
ganache --deterministic

# Сохранение состояния
ganache --database.dbPath ./ganache-db
```

### Ganache GUI

Ganache также доступен как GUI приложение с визуальным интерфейсом:

- Просмотр аккаунтов и балансов
- История транзакций
- Просмотр блоков
- Логи событий
- Настройка параметров сети

Скачать можно с [официального сайта](https://trufflesuite.com/ganache/).

## Truffle Console

Интерактивная консоль для взаимодействия с контрактами:

```bash
# Запуск консоли
truffle console

# На конкретной сети
truffle console --network sepolia
```

```javascript
// В консоли
truffle(development)> const token = await Token.deployed()
truffle(development)> const balance = await token.balanceOf(accounts[0])
truffle(development)> balance.toString()
'1000000000000000000000000'

truffle(development)> await token.transfer(accounts[1], web3.utils.toWei("100", "ether"))
truffle(development)> const newBalance = await token.balanceOf(accounts[1])
truffle(development)> web3.utils.fromWei(newBalance, "ether")
'100'
```

## Truffle Dashboard

Truffle Dashboard позволяет безопасно подписывать транзакции через MetaMask без раскрытия приватных ключей:

```bash
# Запуск Dashboard
truffle dashboard

# В другом терминале - деплой через Dashboard
truffle migrate --network dashboard
```

## Плагины

### truffle-plugin-verify

Верификация контрактов на Etherscan:

```bash
npm install truffle-plugin-verify
```

```javascript
// truffle-config.js
plugins: ['truffle-plugin-verify'],
api_keys: {
  etherscan: 'YOUR_API_KEY'
}
```

```bash
# Верификация после деплоя
truffle run verify Token --network sepolia
```

### truffle-contract-size

Проверка размера контрактов:

```bash
npm install truffle-contract-size
```

```bash
truffle run contract-size
```

## Отладка

### Truffle Debugger

```bash
# Отладка транзакции
truffle debug <tx-hash>

# Команды в дебаггере
# o - step over
# i - step into
# u - step out
# n - step next
# ; - print instruction
# p - print variables
# v - print variables and values
# q - quit
```

## Типичные проблемы и решения

### Проблема: "Migrations" not found

```bash
# Убедитесь, что контракт Migrations.sol существует
truffle migrate --reset
```

### Проблема: Out of gas

```javascript
// truffle-config.js
networks: {
  development: {
    gas: 8000000,
    gasPrice: 20000000000,
  }
}
```

### Проблема: Nonce too low

```bash
# Сбросьте состояние Ganache или используйте --reset
truffle migrate --reset
```

## Статус проекта

> **Внимание:** Truffle Suite официально прекращен (sunset) с декабря 2023 года. Consensys рекомендует переход на Hardhat.

**Что это означает:**
- Новые функции не добавляются
- Критические баги могут не исправляться
- Поддержка сообщества сокращается
- Документация остается доступной

**Рекомендации:**
1. Для новых проектов используйте Hardhat или Foundry
2. Для существующих Truffle проектов рассмотрите миграцию
3. Знание Truffle полезно для работы с legacy проектами

## Миграция на Hardhat

```bash
# Установка Hardhat
npm install --save-dev hardhat

# Инициализация
npx hardhat init

# Импорт Truffle конфигурации
npm install --save-dev @nomiclabs/hardhat-truffle5
```

```javascript
// hardhat.config.js
require("@nomiclabs/hardhat-truffle5");

module.exports = {
  solidity: "0.8.24",
  networks: {
    // Скопируйте настройки из truffle-config.js
  }
};
```

## Ресурсы

- [Документация Truffle](https://trufflesuite.com/docs/)
- [Truffle GitHub](https://github.com/trufflesuite/truffle)
- [Ganache](https://trufflesuite.com/ganache/)
- [Truffle Boxes](https://trufflesuite.com/boxes/)
