# Unit тестирование смарт-контрактов

## Введение

Unit-тестирование (модульное тестирование) — это проверка отдельных функций и методов контракта в изоляции от других компонентов системы. Цель — убедиться, что каждая функция работает корректно при различных входных данных.

## Hardhat тестирование

### Настройка проекта

```bash
# Создание проекта
mkdir my-project && cd my-project
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Инициализация
npx hardhat init
```

### Структура проекта

```
my-project/
├── contracts/
│   └── Token.sol
├── test/
│   └── Token.test.js
├── hardhat.config.js
└── package.json
```

### Базовый тест на JavaScript

```javascript
// test/Token.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Token Contract", function() {
  let token;
  let owner;
  let addr1;
  let addr2;

  beforeEach(async function() {
    // Получаем аккаунты
    [owner, addr1, addr2] = await ethers.getSigners();

    // Деплоим контракт перед каждым тестом
    const Token = await ethers.getContractFactory("Token");
    token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.waitForDeployment();
  });

  describe("Deployment", function() {
    it("should set the right owner", async function() {
      expect(await token.owner()).to.equal(owner.address);
    });

    it("should assign total supply to owner", async function() {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });

    it("should set correct name and symbol", async function() {
      expect(await token.name()).to.equal("MyToken");
      expect(await token.symbol()).to.equal("MTK");
    });
  });

  describe("Transfers", function() {
    it("should transfer tokens between accounts", async function() {
      // Трансфер от owner к addr1
      await token.transfer(addr1.address, 100);
      expect(await token.balanceOf(addr1.address)).to.equal(100);

      // Трансфер от addr1 к addr2
      await token.connect(addr1).transfer(addr2.address, 50);
      expect(await token.balanceOf(addr2.address)).to.equal(50);
      expect(await token.balanceOf(addr1.address)).to.equal(50);
    });

    it("should fail if sender has insufficient balance", async function() {
      const initialBalance = await token.balanceOf(owner.address);

      // Попытка перевести больше, чем есть на балансе
      await expect(
        token.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWith("Insufficient balance");

      // Баланс owner не должен измениться
      expect(await token.balanceOf(owner.address)).to.equal(initialBalance);
    });

    it("should update balances after transfers", async function() {
      const initialOwnerBalance = await token.balanceOf(owner.address);

      await token.transfer(addr1.address, 100);
      await token.transfer(addr2.address, 50);

      expect(await token.balanceOf(owner.address))
        .to.equal(initialOwnerBalance - 150n);
      expect(await token.balanceOf(addr1.address)).to.equal(100);
      expect(await token.balanceOf(addr2.address)).to.equal(50);
    });
  });
});
```

### Тестирование на TypeScript

```typescript
// test/Token.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { Token } from "../typechain-types";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";

describe("Token Contract", function() {
  let token: Token;
  let owner: SignerWithAddress;
  let addr1: SignerWithAddress;

  beforeEach(async function() {
    [owner, addr1] = await ethers.getSigners();

    const TokenFactory = await ethers.getContractFactory("Token");
    token = await TokenFactory.deploy("MyToken", "MTK", 1000000n);
  });

  it("should transfer with correct types", async function() {
    const amount: bigint = 100n;
    await token.transfer(addr1.address, amount);

    const balance: bigint = await token.balanceOf(addr1.address);
    expect(balance).to.equal(amount);
  });
});
```

## Fixtures — оптимизация тестов

Fixtures позволяют переиспользовать состояние между тестами, значительно ускоряя выполнение.

```javascript
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

async function deployTokenFixture() {
  const [owner, addr1, addr2] = await ethers.getSigners();

  const Token = await ethers.getContractFactory("Token");
  const token = await Token.deploy("MyToken", "MTK", 1000000);

  // Дополнительная настройка
  await token.transfer(addr1.address, 1000);

  return { token, owner, addr1, addr2 };
}

describe("Token with Fixtures", function() {
  it("test 1", async function() {
    const { token, owner } = await loadFixture(deployTokenFixture);
    // Каждый тест получает чистое состояние
    expect(await token.balanceOf(owner.address)).to.equal(999000);
  });

  it("test 2", async function() {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    // Состояние из test 1 не сохраняется
    expect(await token.balanceOf(addr1.address)).to.equal(1000);
  });
});
```

## Моки и заглушки

### Мок внешнего контракта

```javascript
// contracts/mocks/MockPriceFeed.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MockPriceFeed {
    int256 private price;
    uint8 private _decimals;

    constructor(int256 _price, uint8 decimals_) {
        price = _price;
        _decimals = decimals_;
    }

    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    ) {
        return (1, price, block.timestamp, block.timestamp, 1);
    }

    function setPrice(int256 _price) external {
        price = _price;
    }

    function decimals() external view returns (uint8) {
        return _decimals;
    }
}
```

```javascript
// test/PriceConsumer.test.js
describe("PriceConsumer", function() {
  let priceConsumer;
  let mockPriceFeed;

  beforeEach(async function() {
    // Деплоим мок
    const MockPriceFeed = await ethers.getContractFactory("MockPriceFeed");
    mockPriceFeed = await MockPriceFeed.deploy(
      200000000000n, // $2000.00000000
      8
    );

    // Деплоим тестируемый контракт с моком
    const PriceConsumer = await ethers.getContractFactory("PriceConsumer");
    priceConsumer = await PriceConsumer.deploy(
      await mockPriceFeed.getAddress()
    );
  });

  it("should return correct ETH price", async function() {
    const price = await priceConsumer.getLatestPrice();
    expect(price).to.equal(200000000000n);
  });

  it("should handle price changes", async function() {
    // Изменяем цену в моке
    await mockPriceFeed.setPrice(250000000000n);

    const newPrice = await priceConsumer.getLatestPrice();
    expect(newPrice).to.equal(250000000000n);
  });
});
```

### Использование Smock для моков

```bash
npm install --save-dev @defi-wonderland/smock
```

```javascript
const { smock } = require("@defi-wonderland/smock");

describe("With Smock", function() {
  it("should mock contract calls", async function() {
    const mockToken = await smock.fake("IERC20");

    // Настраиваем возвращаемое значение
    mockToken.balanceOf.returns(1000);
    mockToken.transfer.returns(true);

    // Используем мок
    expect(await mockToken.balanceOf(owner.address)).to.equal(1000);

    // Проверяем, что функция была вызвана
    expect(mockToken.balanceOf).to.have.been.calledWith(owner.address);
  });
});
```

## Foundry Unit тестирование

### Настройка проекта

```bash
# Создание проекта
forge init my-foundry-project
cd my-foundry-project
```

### Базовый тест на Solidity

```solidity
// test/Token.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Token.sol";

contract TokenTest is Test {
    Token public token;
    address public owner;
    address public user1;
    address public user2;

    function setUp() public {
        owner = address(this);
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");

        token = new Token("MyToken", "MTK", 1000000);
    }

    function test_InitialSupply() public view {
        assertEq(token.totalSupply(), 1000000);
        assertEq(token.balanceOf(owner), 1000000);
    }

    function test_Transfer() public {
        token.transfer(user1, 100);

        assertEq(token.balanceOf(user1), 100);
        assertEq(token.balanceOf(owner), 999900);
    }

    function test_RevertWhen_InsufficientBalance() public {
        vm.prank(user1); // Выполняем следующий вызов от имени user1

        vm.expectRevert("Insufficient balance");
        token.transfer(user2, 100);
    }

    function test_TransferFrom() public {
        // Owner одобряет user1 на трату токенов
        token.approve(user1, 500);

        // user1 переводит токены от owner к user2
        vm.prank(user1);
        token.transferFrom(owner, user2, 300);

        assertEq(token.balanceOf(user2), 300);
        assertEq(token.allowance(owner, user1), 200);
    }
}
```

### Cheatcodes — управление состоянием

```solidity
contract CheatcodesExample is Test {
    function test_Prank() public {
        address alice = makeAddr("alice");

        // Следующий вызов будет от имени alice
        vm.prank(alice);
        // someContract.doSomething();

        // Все последующие вызовы от alice до stopPrank
        vm.startPrank(alice);
        // someContract.call1();
        // someContract.call2();
        vm.stopPrank();
    }

    function test_Deal() public {
        address user = makeAddr("user");

        // Устанавливаем баланс ETH
        vm.deal(user, 100 ether);
        assertEq(user.balance, 100 ether);
    }

    function test_Warp() public {
        // Перемещаемся во времени
        vm.warp(block.timestamp + 1 days);

        // Перемещаемся на определённый блок
        vm.roll(block.number + 100);
    }

    function test_ExpectEmit() public {
        // Ожидаем событие
        vm.expectEmit(true, true, false, true);
        emit Transfer(address(this), user1, 100);

        token.transfer(user1, 100);
    }

    function test_Snapshot() public {
        // Создаём snapshot состояния
        uint256 snapshotId = vm.snapshot();

        // Делаем изменения
        token.transfer(user1, 100);

        // Откатываемся к snapshot
        vm.revertTo(snapshotId);

        // Состояние восстановлено
        assertEq(token.balanceOf(user1), 0);
    }
}
```

### Тестирование событий

```solidity
contract EventTest is Test {
    Token token;

    event Transfer(address indexed from, address indexed to, uint256 value);

    function setUp() public {
        token = new Token("Test", "TST", 1000);
    }

    function test_EmitsTransferEvent() public {
        address to = makeAddr("recipient");

        // Указываем какие indexed параметры проверять
        // (checkTopic1, checkTopic2, checkTopic3, checkData)
        vm.expectEmit(true, true, false, true);

        // Ожидаемое событие
        emit Transfer(address(this), to, 100);

        // Вызов, который должен emit событие
        token.transfer(to, 100);
    }
}
```

## Best Practices

### 1. Изолированные тесты

```javascript
// Плохо — тесты зависят друг от друга
describe("Bad", function() {
  let balance;

  it("deposits", async function() {
    await vault.deposit(100);
    balance = 100;
  });

  it("withdraws", async function() {
    await vault.withdraw(balance); // Зависит от предыдущего теста!
  });
});

// Хорошо — каждый тест независим
describe("Good", function() {
  beforeEach(async function() {
    await vault.deposit(100);
  });

  it("deposits correctly", async function() {
    expect(await vault.balance()).to.equal(100);
  });

  it("withdraws correctly", async function() {
    await vault.withdraw(100);
    expect(await vault.balance()).to.equal(0);
  });
});
```

### 2. Тестируйте граничные случаи

```javascript
describe("Edge Cases", function() {
  it("handles zero amount", async function() {
    await expect(token.transfer(addr1.address, 0))
      .to.be.revertedWith("Amount must be positive");
  });

  it("handles max uint256", async function() {
    await expect(token.transfer(addr1.address, ethers.MaxUint256))
      .to.be.revertedWith("Insufficient balance");
  });

  it("handles zero address", async function() {
    await expect(token.transfer(ethers.ZeroAddress, 100))
      .to.be.revertedWith("Invalid recipient");
  });
});
```

### 3. Используйте описательные имена

```solidity
// Foundry naming convention
function test_Transfer_RevertsWhen_RecipientIsZeroAddress() public {
    vm.expectRevert("Invalid recipient");
    token.transfer(address(0), 100);
}

function test_Transfer_UpdatesBalances_WhenAmountIsValid() public {
    token.transfer(user1, 100);
    assertEq(token.balanceOf(user1), 100);
}
```

### 4. Группируйте связанные тесты

```javascript
describe("Token", function() {
  describe("Deployment", function() { /* ... */ });

  describe("Transfers", function() {
    describe("when sender has enough balance", function() { /* ... */ });
    describe("when sender doesn't have enough balance", function() { /* ... */ });
  });

  describe("Allowances", function() {
    describe("approve", function() { /* ... */ });
    describe("transferFrom", function() { /* ... */ });
  });
});
```

## Запуск тестов

### Hardhat

```bash
# Все тесты
npx hardhat test

# Конкретный файл
npx hardhat test test/Token.test.js

# С фильтром по имени
npx hardhat test --grep "transfer"

# С отчётом о газе
REPORT_GAS=true npx hardhat test
```

### Foundry

```bash
# Все тесты
forge test

# С детализацией
forge test -vvvv

# Конкретный тест
forge test --match-test test_Transfer

# Конкретный контракт
forge test --match-contract TokenTest

# Отчёт о газе
forge test --gas-report
```
