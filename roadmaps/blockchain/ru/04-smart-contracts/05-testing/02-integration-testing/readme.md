# Интеграционное тестирование смарт-контрактов

## Введение

Интеграционное тестирование проверяет корректность взаимодействия между несколькими смарт-контрактами и внешними системами. В отличие от unit-тестов, которые проверяют отдельные функции в изоляции, интеграционные тесты проверяют полные сценарии использования протокола.

## Зачем нужны интеграционные тесты

```
┌─────────────────────────────────────────────────────────────────┐
│                    DeFi Protocol Architecture                    │
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Token   │───▶│  Vault   │───▶│ Strategy │───▶│ External │  │
│  │ Contract │    │ Contract │    │ Contract │    │   DEX    │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │              │                │               │         │
│       └──────────────┴────────────────┴───────────────┘         │
│                    Интеграционные тесты                          │
│             проверяют всю цепочку взаимодействий                 │
└─────────────────────────────────────────────────────────────────┘
```

**Что проверяют интеграционные тесты:**
- Корректность передачи данных между контрактами
- Правильность расчётов с учётом всех комиссий
- Обработку edge cases на уровне протокола
- Взаимодействие с внешними протоколами (Uniswap, Aave, Chainlink)

## Mainnet Fork тестирование

### Концепция форка

Mainnet fork создаёт локальную копию состояния основной сети на определённом блоке. Это позволяет:
- Тестировать взаимодействие с реальными протоколами
- Использовать реальные данные (цены, ликвидность)
- Проверять сценарии с существующими токенами

### Настройка Hardhat Fork

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: "0.8.20",
  networks: {
    hardhat: {
      forking: {
        url: process.env.MAINNET_RPC_URL,
        blockNumber: 18500000, // Фиксируем блок для воспроизводимости
        enabled: true
      }
    },
    // Отдельная сеть для форка с другими параметрами
    mainnetFork: {
      url: "http://127.0.0.1:8545",
      forking: {
        url: process.env.MAINNET_RPC_URL,
        blockNumber: 18500000
      }
    }
  }
};
```

### Настройка Foundry Fork

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"

[profile.fork]
fork_url = "${MAINNET_RPC_URL}"
fork_block_number = 18500000
```

## Тестирование взаимодействия с DEX

### Интеграция с Uniswap V3

```javascript
// test/integration/UniswapIntegration.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

// Адреса mainnet контрактов
const UNISWAP_ROUTER = "0xE592427A0AEce92De3Edee1F18E0157C05861564";
const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
const USDC = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
const WETH_USDC_POOL = "0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640";

describe("Uniswap V3 Integration", function() {
  let swapRouter;
  let weth;
  let usdc;
  let whale;
  let user;

  before(async function() {
    // Получаем контракты по адресам
    swapRouter = await ethers.getContractAt(
      "ISwapRouter",
      UNISWAP_ROUTER
    );
    weth = await ethers.getContractAt("IWETH", WETH);
    usdc = await ethers.getContractAt("IERC20", USDC);

    [user] = await ethers.getSigners();

    // Impersonate whale account для получения токенов
    const whaleAddress = "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503";
    await ethers.provider.send("hardhat_impersonateAccount", [whaleAddress]);
    whale = await ethers.getSigner(whaleAddress);

    // Отправляем ETH whale для газа
    await user.sendTransaction({
      to: whaleAddress,
      value: ethers.parseEther("10")
    });
  });

  it("should swap WETH for USDC", async function() {
    const amountIn = ethers.parseEther("1");

    // Депозит ETH в WETH
    await weth.connect(user).deposit({ value: amountIn });

    // Approve router
    await weth.connect(user).approve(UNISWAP_ROUTER, amountIn);

    const usdcBalanceBefore = await usdc.balanceOf(user.address);

    // Выполняем swap
    const params = {
      tokenIn: WETH,
      tokenOut: USDC,
      fee: 500, // 0.05% pool
      recipient: user.address,
      deadline: Math.floor(Date.now() / 1000) + 3600,
      amountIn: amountIn,
      amountOutMinimum: 0,
      sqrtPriceLimitX96: 0
    };

    await swapRouter.connect(user).exactInputSingle(params);

    const usdcBalanceAfter = await usdc.balanceOf(user.address);
    const usdcReceived = usdcBalanceAfter - usdcBalanceBefore;

    console.log(`Received ${ethers.formatUnits(usdcReceived, 6)} USDC`);

    // USDC должен быть получен (примерно $2000 за 1 ETH)
    expect(usdcReceived).to.be.gt(0);
  });
});
```

### Foundry Fork тест

```solidity
// test/integration/UniswapIntegration.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/interfaces/ISwapRouter.sol";
import "../src/interfaces/IERC20.sol";
import "../src/interfaces/IWETH.sol";

contract UniswapIntegrationTest is Test {
    ISwapRouter constant ROUTER = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
    IWETH constant WETH = IWETH(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    IERC20 constant USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);

    address user;

    function setUp() public {
        // Создаём fork mainnet
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), 18500000);

        user = makeAddr("user");
        vm.deal(user, 100 ether);
    }

    function test_SwapWETHForUSDC() public {
        uint256 amountIn = 1 ether;

        vm.startPrank(user);

        // Wrap ETH
        WETH.deposit{value: amountIn}();
        WETH.approve(address(ROUTER), amountIn);

        uint256 usdcBefore = USDC.balanceOf(user);

        // Swap
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: address(WETH),
            tokenOut: address(USDC),
            fee: 500,
            recipient: user,
            deadline: block.timestamp + 3600,
            amountIn: amountIn,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        });

        ROUTER.exactInputSingle(params);

        uint256 usdcAfter = USDC.balanceOf(user);
        uint256 usdcReceived = usdcAfter - usdcBefore;

        console.log("USDC received:", usdcReceived / 1e6);
        assertGt(usdcReceived, 1000e6); // Минимум $1000

        vm.stopPrank();
    }
}
```

## Тестирование сложных DeFi сценариев

### Полный флоу lending протокола

```javascript
// test/integration/LendingFlow.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture, time } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

describe("Lending Protocol Integration", function() {
  async function deployProtocolFixture() {
    const [owner, lender, borrower] = await ethers.getSigners();

    // Deploy tokens
    const Token = await ethers.getContractFactory("MockERC20");
    const collateralToken = await Token.deploy("Collateral", "COL", 18);
    const borrowToken = await Token.deploy("Borrow", "BOR", 18);

    // Deploy price oracle
    const Oracle = await ethers.getContractFactory("MockOracle");
    const oracle = await Oracle.deploy();
    await oracle.setPrice(await collateralToken.getAddress(), ethers.parseEther("100"));
    await oracle.setPrice(await borrowToken.getAddress(), ethers.parseEther("1"));

    // Deploy lending pool
    const LendingPool = await ethers.getContractFactory("LendingPool");
    const lendingPool = await LendingPool.deploy(await oracle.getAddress());

    // Configure pool
    await lendingPool.addAsset(
      await collateralToken.getAddress(),
      7500 // 75% LTV
    );
    await lendingPool.addAsset(
      await borrowToken.getAddress(),
      8000 // 80% LTV
    );

    // Mint tokens
    await collateralToken.mint(borrower.address, ethers.parseEther("1000"));
    await borrowToken.mint(lender.address, ethers.parseEther("100000"));

    return {
      lendingPool, collateralToken, borrowToken, oracle,
      owner, lender, borrower
    };
  }

  describe("Full Lending Cycle", function() {
    it("should complete deposit -> borrow -> repay -> withdraw cycle", async function() {
      const {
        lendingPool, collateralToken, borrowToken,
        lender, borrower
      } = await loadFixture(deployProtocolFixture);

      // Step 1: Lender deposits borrow tokens
      const depositAmount = ethers.parseEther("10000");
      await borrowToken.connect(lender)
        .approve(await lendingPool.getAddress(), depositAmount);
      await lendingPool.connect(lender)
        .deposit(await borrowToken.getAddress(), depositAmount);

      expect(await lendingPool.getDepositBalance(
        lender.address,
        await borrowToken.getAddress()
      )).to.equal(depositAmount);

      // Step 2: Borrower deposits collateral
      const collateralAmount = ethers.parseEther("100"); // Worth $10,000
      await collateralToken.connect(borrower)
        .approve(await lendingPool.getAddress(), collateralAmount);
      await lendingPool.connect(borrower)
        .deposit(await collateralToken.getAddress(), collateralAmount);

      // Step 3: Borrower takes a loan
      // Max borrow = $10,000 * 75% = $7,500
      const borrowAmount = ethers.parseEther("7000"); // $7,000 worth
      await lendingPool.connect(borrower)
        .borrow(await borrowToken.getAddress(), borrowAmount);

      expect(await borrowToken.balanceOf(borrower.address))
        .to.equal(borrowAmount);

      // Step 4: Time passes, interest accrues
      await time.increase(365 * 24 * 60 * 60); // 1 year

      // Step 5: Borrower repays with interest
      const debt = await lendingPool.getBorrowBalance(
        borrower.address,
        await borrowToken.getAddress()
      );
      console.log(`Total debt after 1 year: ${ethers.formatEther(debt)}`);

      await borrowToken.connect(borrower)
        .approve(await lendingPool.getAddress(), debt);
      await lendingPool.connect(borrower)
        .repay(await borrowToken.getAddress(), debt);

      // Step 6: Borrower withdraws collateral
      await lendingPool.connect(borrower)
        .withdraw(await collateralToken.getAddress(), collateralAmount);

      expect(await collateralToken.balanceOf(borrower.address))
        .to.equal(ethers.parseEther("1000"));

      // Step 7: Lender withdraws with interest
      const lenderBalance = await lendingPool.getDepositBalance(
        lender.address,
        await borrowToken.getAddress()
      );
      console.log(`Lender balance with interest: ${ethers.formatEther(lenderBalance)}`);

      await lendingPool.connect(lender)
        .withdraw(await borrowToken.getAddress(), lenderBalance);

      expect(lenderBalance).to.be.gt(depositAmount);
    });

    it("should liquidate undercollateralized position", async function() {
      const {
        lendingPool, collateralToken, borrowToken, oracle,
        owner, lender, borrower
      } = await loadFixture(deployProtocolFixture);

      // Setup: lender deposits, borrower borrows at max
      const depositAmount = ethers.parseEther("10000");
      await borrowToken.connect(lender)
        .approve(await lendingPool.getAddress(), depositAmount);
      await lendingPool.connect(lender)
        .deposit(await borrowToken.getAddress(), depositAmount);

      const collateralAmount = ethers.parseEther("100");
      await collateralToken.connect(borrower)
        .approve(await lendingPool.getAddress(), collateralAmount);
      await lendingPool.connect(borrower)
        .deposit(await collateralToken.getAddress(), collateralAmount);

      const borrowAmount = ethers.parseEther("7000");
      await lendingPool.connect(borrower)
        .borrow(await borrowToken.getAddress(), borrowAmount);

      // Price drops - collateral now worth $5,000 (50% drop)
      await oracle.setPrice(
        await collateralToken.getAddress(),
        ethers.parseEther("50")
      );

      // Check health factor
      const healthFactor = await lendingPool.getHealthFactor(borrower.address);
      console.log(`Health factor after price drop: ${ethers.formatEther(healthFactor)}`);
      expect(healthFactor).to.be.lt(ethers.parseEther("1")); // Undercollateralized

      // Liquidator liquidates
      const liquidationAmount = ethers.parseEther("3500"); // 50% of debt
      await borrowToken.mint(owner.address, liquidationAmount);
      await borrowToken.approve(await lendingPool.getAddress(), liquidationAmount);

      await lendingPool.liquidate(
        borrower.address,
        await borrowToken.getAddress(),
        liquidationAmount,
        await collateralToken.getAddress()
      );

      // Verify liquidation happened
      const remainingDebt = await lendingPool.getBorrowBalance(
        borrower.address,
        await borrowToken.getAddress()
      );
      expect(remainingDebt).to.be.lt(borrowAmount);
    });
  });
});
```

## Тестирование с Chainlink

```javascript
// test/integration/ChainlinkIntegration.test.js
describe("Chainlink Price Feed Integration", function() {
  const ETH_USD_FEED = "0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419";

  it("should get real ETH price from Chainlink", async function() {
    const priceFeed = await ethers.getContractAt(
      "AggregatorV3Interface",
      ETH_USD_FEED
    );

    const [, answer, , updatedAt,] = await priceFeed.latestRoundData();
    const decimals = await priceFeed.decimals();

    const price = Number(answer) / 10 ** Number(decimals);
    console.log(`ETH/USD Price: $${price}`);
    console.log(`Last updated: ${new Date(Number(updatedAt) * 1000)}`);

    expect(price).to.be.gt(0);
  });
});
```

## Impersonation - тестирование от имени whale

```javascript
describe("Whale Impersonation", function() {
  const USDC_WHALE = "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503";

  it("should transfer USDC from whale", async function() {
    const usdc = await ethers.getContractAt(
      "IERC20",
      "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
    );
    const [receiver] = await ethers.getSigners();

    // Impersonate whale
    await ethers.provider.send("hardhat_impersonateAccount", [USDC_WHALE]);
    const whale = await ethers.getSigner(USDC_WHALE);

    // Fund whale with ETH for gas
    await receiver.sendTransaction({
      to: USDC_WHALE,
      value: ethers.parseEther("1")
    });

    const amount = 1000000n * 10n ** 6n; // 1M USDC
    const balanceBefore = await usdc.balanceOf(receiver.address);

    await usdc.connect(whale).transfer(receiver.address, amount);

    expect(await usdc.balanceOf(receiver.address))
      .to.equal(balanceBefore + amount);

    // Stop impersonation
    await ethers.provider.send("hardhat_stopImpersonatingAccount", [USDC_WHALE]);
  });
});
```

## Foundry Integration тесты

```solidity
// test/integration/DeFiIntegration.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

interface IAave {
    function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}

contract AaveIntegrationTest is Test {
    IAave constant AAVE_POOL = IAave(0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2);
    IERC20 constant USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    IERC20 constant aUSDC = IERC20(0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c);

    address whale = 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503;
    address user;

    function setUp() public {
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"));
        user = makeAddr("user");

        // Transfer USDC from whale to user
        vm.prank(whale);
        USDC.transfer(user, 10000e6); // 10,000 USDC
    }

    function test_DepositAndWithdrawFromAave() public {
        uint256 depositAmount = 5000e6; // 5,000 USDC

        vm.startPrank(user);

        // Approve and deposit
        USDC.approve(address(AAVE_POOL), depositAmount);
        AAVE_POOL.deposit(address(USDC), depositAmount, user, 0);

        // Check aToken balance
        uint256 aBalance = aUSDC.balanceOf(user);
        assertGt(aBalance, 0);
        console.log("aUSDC received:", aBalance / 1e6);

        // Time travel for interest accrual
        vm.warp(block.timestamp + 365 days);

        // Withdraw all
        uint256 withdrawn = AAVE_POOL.withdraw(address(USDC), type(uint256).max, user);
        console.log("USDC withdrawn:", withdrawn / 1e6);

        // Should have earned interest
        assertGt(withdrawn, depositAmount);

        vm.stopPrank();
    }

    function test_FlashLoan() public {
        // Flash loan тест будет здесь
    }
}
```

## Организация интеграционных тестов

```
test/
├── unit/
│   ├── Token.test.js
│   └── Vault.test.js
├── integration/
│   ├── fixtures/
│   │   └── deployFixture.js
│   ├── uniswap/
│   │   ├── SwapExactInput.test.js
│   │   └── AddLiquidity.test.js
│   ├── aave/
│   │   ├── Deposit.test.js
│   │   └── FlashLoan.test.js
│   └── full-protocol/
│       └── E2EFlow.test.js
└── fork/
    └── MainnetScenarios.test.js
```

## Best Practices

### 1. Фиксируйте номер блока

```javascript
// Для воспроизводимости результатов
forking: {
  url: process.env.MAINNET_RPC_URL,
  blockNumber: 18500000 // Всегда фиксированный блок
}
```

### 2. Кэшируйте fork данные

```bash
# Hardhat
npx hardhat node --fork <RPC_URL> --fork-block-number 18500000

# Foundry
forge test --fork-url $RPC --fork-block-number 18500000 --fork-cache-path cache/fork
```

### 3. Тестируйте реалистичные сценарии

```javascript
it("should handle real-world slippage", async function() {
  // Используем реалистичные суммы
  const amount = ethers.parseEther("1000"); // Не 1 wei

  // Учитываем slippage
  const minAmountOut = expectedAmount * 995n / 1000n; // 0.5% slippage

  await router.swap(tokenIn, tokenOut, amount, minAmountOut);
});
```

### 4. Изолируйте fork тесты

```javascript
// package.json
{
  "scripts": {
    "test": "hardhat test test/unit/**/*.test.js",
    "test:integration": "hardhat test test/integration/**/*.test.js",
    "test:fork": "FORK=true hardhat test test/fork/**/*.test.js"
  }
}
```
