# Lending и Borrowing протоколы

## Введение

**Lending/Borrowing протоколы** — это DeFi-приложения, позволяющие пользователям давать криптовалюту в долг (lending) и занимать её (borrowing) без посредников. Все операции выполняются смарт-контрактами.

## Как работает DeFi Lending

```
┌─────────────────────────────────────────────────────────────┐
│                   Lending Protocol                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Lenders                    Borrowers                      │
│   ┌─────┐                    ┌─────┐                       │
│   │ ETH │ ─────► ┌──────┐ ◄─── │ DAI │ (Collateral)        │
│   └─────┘        │ Pool │      └─────┘                       │
│                  └──────┘                                   │
│   Получают           │        Получают                      │
│   aTokens            │        кредит                        │
│   + проценты         │        + платят проценты             │
│                      ▼                                      │
│              Supply Rate (APY для lenders)                  │
│              Borrow Rate (APY для borrowers)                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Ключевые концепции

### 1. Overcollateralization (Избыточное обеспечение)

В отличие от традиционных кредитов, DeFi требует залог больше, чем сумма кредита.

```solidity
// Пример: LTV (Loan-to-Value) = 75%
// Если депозит ETH на $1000, можно занять максимум $750

contract CollateralCalculator {
    uint256 public constant LTV = 75; // 75%
    uint256 public constant LIQUIDATION_THRESHOLD = 80; // 80%

    function maxBorrow(uint256 collateralValue) public pure returns (uint256) {
        return (collateralValue * LTV) / 100;
    }

    function healthFactor(
        uint256 collateralValue,
        uint256 borrowedValue
    ) public pure returns (uint256) {
        // Health Factor = (Collateral * Liquidation Threshold) / Borrowed
        // Health Factor < 1 = ликвидация
        return (collateralValue * LIQUIDATION_THRESHOLD) / (borrowedValue * 100);
    }
}
```

### 2. Health Factor

Показатель здоровья позиции. Если падает ниже 1, позиция ликвидируется.

```
Health Factor = (Collateral Value × Liquidation Threshold) / Total Borrows

Пример:
- Залог: 10 ETH × $2000 = $20,000
- Liquidation Threshold: 80%
- Занято: $12,000

Health Factor = (20,000 × 0.80) / 12,000 = 1.33 ✅

Если ETH упадет до $1500:
Health Factor = (15,000 × 0.80) / 12,000 = 1.00 ⚠️ (граница ликвидации)
```

### 3. Utilization Rate

Процент использования пула.

```solidity
// Utilization Rate = Total Borrows / Total Supply

library InterestRateModel {
    uint256 constant OPTIMAL_UTILIZATION = 80e16; // 80%
    uint256 constant BASE_RATE = 2e16;            // 2%
    uint256 constant SLOPE1 = 4e16;               // 4%
    uint256 constant SLOPE2 = 75e16;              // 75%

    function calculateBorrowRate(
        uint256 totalSupply,
        uint256 totalBorrows
    ) public pure returns (uint256) {
        if (totalSupply == 0) return BASE_RATE;

        uint256 utilization = (totalBorrows * 1e18) / totalSupply;

        if (utilization <= OPTIMAL_UTILIZATION) {
            // Линейный рост до оптимальной утилизации
            return BASE_RATE + (utilization * SLOPE1) / OPTIMAL_UTILIZATION;
        } else {
            // Резкий рост после оптимальной утилизации
            uint256 excessUtilization = utilization - OPTIMAL_UTILIZATION;
            uint256 maxExcess = 1e18 - OPTIMAL_UTILIZATION;
            return BASE_RATE + SLOPE1 + (excessUtilization * SLOPE2) / maxExcess;
        }
    }

    function calculateSupplyRate(
        uint256 borrowRate,
        uint256 utilization,
        uint256 reserveFactor
    ) public pure returns (uint256) {
        // Supply Rate = Borrow Rate × Utilization × (1 - Reserve Factor)
        return (borrowRate * utilization * (1e18 - reserveFactor)) / 1e36;
    }
}
```

## Механизм ликвидации

### Когда происходит ликвидация

Ликвидация происходит, когда Health Factor падает ниже 1.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LiquidationExample {
    struct Position {
        uint256 collateralAmount;  // в ETH
        uint256 borrowedAmount;    // в DAI
        uint256 collateralPrice;   // цена ETH в DAI
    }

    uint256 public constant LIQUIDATION_THRESHOLD = 80; // 80%
    uint256 public constant LIQUIDATION_BONUS = 5;       // 5% бонус ликвидатору
    uint256 public constant CLOSE_FACTOR = 50;           // можно закрыть 50% долга

    function isLiquidatable(Position memory pos) public pure returns (bool) {
        uint256 collateralValue = pos.collateralAmount * pos.collateralPrice;
        uint256 maxBorrow = (collateralValue * LIQUIDATION_THRESHOLD) / 100;
        return pos.borrowedAmount > maxBorrow;
    }

    function liquidate(
        Position storage pos,
        uint256 repayAmount
    ) external {
        require(isLiquidatable(pos), "Position is healthy");

        // Максимум можно погасить CLOSE_FACTOR% долга
        uint256 maxRepay = (pos.borrowedAmount * CLOSE_FACTOR) / 100;
        require(repayAmount <= maxRepay, "Too much repay");

        // Ликвидатор получает залог + бонус
        uint256 collateralToSeize = repayAmount / pos.collateralPrice;
        uint256 bonus = (collateralToSeize * LIQUIDATION_BONUS) / 100;
        uint256 totalSeized = collateralToSeize + bonus;

        // Обновляем позицию
        pos.borrowedAmount -= repayAmount;
        pos.collateralAmount -= totalSeized;

        // Перевод токенов (упрощенно)
        // 1. Ликвидатор платит repayAmount в DAI
        // 2. Ликвидатор получает totalSeized в ETH
    }
}
```

### Ликвидационный бот

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@aave/v3-core/contracts/interfaces/IPool.sol";
import "@aave/v3-core/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";

contract LiquidationBot is FlashLoanSimpleReceiverBase {

    constructor(IPoolAddressesProvider provider)
        FlashLoanSimpleReceiverBase(provider) {}

    function liquidateWithFlashLoan(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover
    ) external {
        // Берем flash loan для погашения долга
        POOL.flashLoanSimple(
            address(this),
            debtAsset,
            debtToCover,
            abi.encode(collateralAsset, user),
            0
        );
    }

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        (address collateralAsset, address user) = abi.decode(
            params,
            (address, address)
        );

        // 1. Погашаем долг пользователя, получаем его залог
        IERC20(asset).approve(address(POOL), amount);
        POOL.liquidationCall(
            collateralAsset,
            asset,
            user,
            amount,
            false // receiveAToken = false, получаем underlying
        );

        // 2. Свапаем залог обратно в debt asset
        uint256 collateralReceived = IERC20(collateralAsset).balanceOf(address(this));
        _swapToRepay(collateralAsset, asset, collateralReceived, amount + premium);

        // 3. Возвращаем flash loan + premium
        IERC20(asset).approve(address(POOL), amount + premium);

        // 4. Профит остается на контракте
        return true;
    }

    function _swapToRepay(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOutMin
    ) internal {
        // Логика свапа через DEX
    }
}

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}
```

## Основные протоколы

### Aave

Крупнейший lending протокол с поддержкой множества сетей.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@aave/v3-core/contracts/interfaces/IPool.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract AaveIntegration {
    IPool public immutable aavePool;

    constructor(address _pool) {
        aavePool = IPool(_pool);
    }

    // Депозит в Aave
    function deposit(address asset, uint256 amount) external {
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        IERC20(asset).approve(address(aavePool), amount);

        aavePool.supply(
            asset,
            amount,
            msg.sender,  // onBehalfOf - aTokens идут пользователю
            0            // referralCode
        );
    }

    // Заем из Aave
    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode // 1 = stable, 2 = variable
    ) external {
        aavePool.borrow(
            asset,
            amount,
            interestRateMode,
            0,           // referralCode
            msg.sender   // onBehalfOf
        );

        // Переводим занятые токены пользователю
        IERC20(asset).transfer(msg.sender, amount);
    }

    // Погашение долга
    function repay(
        address asset,
        uint256 amount,
        uint256 interestRateMode
    ) external {
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        IERC20(asset).approve(address(aavePool), amount);

        aavePool.repay(
            asset,
            amount,
            interestRateMode,
            msg.sender
        );
    }

    // Вывод депозита
    function withdraw(address asset, uint256 amount) external {
        aavePool.withdraw(
            asset,
            amount,
            msg.sender
        );
    }

    // Получить данные аккаунта
    function getAccountData(address user) external view returns (
        uint256 totalCollateralBase,
        uint256 totalDebtBase,
        uint256 availableBorrowsBase,
        uint256 currentLiquidationThreshold,
        uint256 ltv,
        uint256 healthFactor
    ) {
        return aavePool.getUserAccountData(user);
    }
}
```

### Compound

Пионер DeFi lending с моделью cTokens.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICToken {
    function mint(uint256 mintAmount) external returns (uint256);
    function redeem(uint256 redeemTokens) external returns (uint256);
    function redeemUnderlying(uint256 redeemAmount) external returns (uint256);
    function borrow(uint256 borrowAmount) external returns (uint256);
    function repayBorrow(uint256 repayAmount) external returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
    function exchangeRateCurrent() external returns (uint256);
    function borrowBalanceCurrent(address account) external returns (uint256);
}

interface IComptroller {
    function enterMarkets(address[] calldata cTokens) external returns (uint256[] memory);
    function getAccountLiquidity(address account) external view returns (uint256, uint256, uint256);
}

contract CompoundIntegration {
    ICToken public cToken;
    IComptroller public comptroller;
    IERC20 public underlying;

    constructor(address _cToken, address _comptroller, address _underlying) {
        cToken = ICToken(_cToken);
        comptroller = IComptroller(_comptroller);
        underlying = IERC20(_underlying);
    }

    // Депозит - получаем cTokens
    function supply(uint256 amount) external {
        underlying.transferFrom(msg.sender, address(this), amount);
        underlying.approve(address(cToken), amount);

        // mint возвращает 0 при успехе
        require(cToken.mint(amount) == 0, "Mint failed");

        // Переводим cTokens пользователю
        uint256 cTokenBalance = cToken.balanceOf(address(this));
        // В реальности используем transfer
    }

    // Включить как залог
    function enableAsCollateral() external {
        address[] memory cTokens = new address[](1);
        cTokens[0] = address(cToken);
        comptroller.enterMarkets(cTokens);
    }

    // Проверить ликвидность
    function checkLiquidity(address account) external view returns (
        uint256 error,
        uint256 liquidity,
        uint256 shortfall
    ) {
        return comptroller.getAccountLiquidity(account);
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

### MakerDAO

Создатель DAI — децентрализованного стейблкоина.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Упрощенный интерфейс MakerDAO Vault
interface IDssCdpManager {
    function open(bytes32 ilk, address usr) external returns (uint256 cdp);
    function frob(uint256 cdp, int256 dink, int256 dart) external;
}

interface IGemJoin {
    function join(address usr, uint256 wad) external;
    function exit(address usr, uint256 wad) external;
}

interface IDaiJoin {
    function join(address usr, uint256 wad) external;
    function exit(address usr, uint256 wad) external;
}

contract MakerVaultExample {
    IDssCdpManager public manager;
    IGemJoin public ethJoin;
    IDaiJoin public daiJoin;

    // Создание Vault и генерация DAI
    function openVaultAndDrawDai(
        uint256 ethAmount,
        uint256 daiAmount
    ) external payable {
        require(msg.value == ethAmount, "Wrong ETH amount");

        // 1. Открыть новый CDP
        uint256 cdpId = manager.open("ETH-A", address(this));

        // 2. Депозит ETH как залог
        ethJoin.join(address(this), ethAmount);

        // 3. Сгенерировать DAI
        // dink = изменение залога (положительное = добавить)
        // dart = изменение долга (положительное = занять)
        manager.frob(
            cdpId,
            int256(ethAmount),  // dink
            int256(daiAmount)   // dart
        );

        // 4. Вывести DAI
        daiJoin.exit(msg.sender, daiAmount);
    }
}
```

## Стратегии использования

### 1. Leverage Long

Увеличение позиции через повторное залоговое кредитование.

```solidity
// Пример: 3x Leverage на ETH
// 1. Депозит 1 ETH
// 2. Занять USDC (75% LTV = $1500 при ETH=$2000)
// 3. Купить ETH на USDC
// 4. Депозит нового ETH
// 5. Повторить

contract LeverageStrategy {
    function leverageLong(uint256 loops) external payable {
        for (uint256 i = 0; i < loops; i++) {
            // 1. Депозит ETH
            // 2. Занять стейблкоин
            // 3. Swap в ETH
            // 4. Повторить
        }
        // Результат: больше ETH exposure, но и больше риск ликвидации
    }
}
```

### 2. Leverage Short

Ставка на падение цены актива.

```
1. Депозит USDC как залог
2. Занять ETH
3. Продать ETH за USDC
4. Если ETH падает, выкупить дешевле и вернуть долг
```

### 3. Yield на стейблкоинах

Низкорисковый доход на стейблкоинах.

```solidity
contract StablecoinYield {
    function depositStablecoins(uint256 amount) external {
        // Депозит USDC в Aave
        // Получаем 2-5% APY
        // Низкий риск ликвидации (стейблкоины не волатильны)
    }
}
```

## Риски Lending протоколов

### 1. Риск ликвидации

```solidity
// Мониторинг Health Factor
contract HealthMonitor {
    uint256 public constant SAFE_HF = 1.5e18;     // Безопасный уровень
    uint256 public constant WARNING_HF = 1.2e18;  // Предупреждение
    uint256 public constant DANGER_HF = 1.05e18;  // Опасный уровень

    function checkHealth(uint256 healthFactor) public pure returns (string memory) {
        if (healthFactor >= SAFE_HF) return "SAFE";
        if (healthFactor >= WARNING_HF) return "WARNING";
        if (healthFactor >= DANGER_HF) return "DANGER";
        return "LIQUIDATION IMMINENT";
    }
}
```

### 2. Риск оракулов

Манипуляция ценами может привести к неправильным ликвидациям или эксплойтам.

### 3. Smart Contract Risk

Уязвимости в коде протокола.

### 4. Риск процентной ставки

Variable rate может резко вырасти при высокой утилизации.

## Сравнение протоколов

| Характеристика | Aave | Compound | MakerDAO |
|---------------|------|----------|----------|
| Тип | Money Market | Money Market | CDP |
| Токен | AAVE | COMP | MKR |
| Stable Rate | Да | Нет | Stability Fee |
| Flash Loans | Да | Нет | Нет |
| Сети | Multi-chain | Ethereum + L2 | Ethereum |
| Особенность | Isolation Mode | cTokens | DAI minting |

## Best Practices

1. **Держите Health Factor выше 1.5** — запас на волатильность
2. **Используйте стейблкоины как залог** — меньше риск ликвидации
3. **Мониторьте позиции** — используйте DeBank, Zapper
4. **Диверсифицируйте** — не держите всё в одном протоколе
5. **Понимайте процентные ставки** — variable vs stable
6. **Имейте план выхода** — как закрыть позицию при падении рынка

## Полезные инструменты

- [DeBank](https://debank.com/) — мониторинг позиций
- [Zapper](https://zapper.fi/) — управление DeFi портфелем
- [DeFi Saver](https://defisaver.com/) — автоматизация и защита от ликвидации
- [Aave Dashboard](https://app.aave.com/) — официальный интерфейс Aave
