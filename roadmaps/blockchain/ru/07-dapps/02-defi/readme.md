# DeFi (Децентрализованные финансы)

## Что такое DeFi?

**DeFi (Decentralized Finance)** — это экосистема финансовых приложений, построенных на блокчейне, которые работают без посредников (банков, брокеров, бирж). Все операции выполняются смарт-контрактами, код которых открыт и проверяем.

## Ключевые принципы DeFi

### 1. Децентрализация
- Нет центрального органа управления
- Протоколы управляются DAO (Decentralized Autonomous Organization)
- Любой может участвовать без KYC/AML

### 2. Прозрачность
- Весь код смарт-контрактов открыт
- Все транзакции видны в блокчейне
- Аудиты безопасности публичны

### 3. Композируемость (Money Legos)
- Протоколы могут взаимодействовать друг с другом
- Можно строить сложные финансовые стратегии
- Один протокол использует ликвидность другого

### 4. Permissionless (Без разрешений)
- Любой может создать приложение
- Любой может использовать протокол
- Нет географических ограничений

## Основные категории DeFi

```
┌─────────────────────────────────────────────────────────────┐
│                      DeFi Ecosystem                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Lending   │  │     DEX     │  │  Yield      │         │
│  │  Borrowing  │  │    AMM      │  │  Farming    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Stablecoins │  │ Derivatives │  │  Insurance  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Bridges   │  │  Oracles    │  │ Aggregators │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## Ключевые метрики DeFi

### TVL (Total Value Locked)
Общая стоимость активов, заблокированных в протоколе.

```solidity
// Пример расчета TVL для lending протокола
contract LendingPool {
    mapping(address => uint256) public deposits;

    function getTVL() public view returns (uint256) {
        uint256 totalValue = 0;
        // Суммируем все депозиты во всех токенах
        // Конвертируем в USD через oracle
        return totalValue;
    }
}
```

### APY vs APR
- **APR (Annual Percentage Rate)** — простой процент без капитализации
- **APY (Annual Percentage Yield)** — сложный процент с учетом капитализации

```solidity
// Формула APY из APR
// APY = (1 + APR/n)^n - 1
// где n — количество периодов капитализации в год

library APYCalculator {
    function aprToApy(uint256 apr, uint256 compoundingPeriods)
        public pure returns (uint256)
    {
        // apr в базисных пунктах (10000 = 100%)
        // Упрощенная формула для малых APR
        uint256 periodicRate = apr / compoundingPeriods;
        uint256 apy = apr + (apr * apr) / (2 * 10000);
        return apy;
    }
}
```

## Flash Loans (Мгновенные кредиты)

Уникальная возможность DeFi — взять кредит без залога, если вернуть в той же транзакции.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@aave/v3-core/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@aave/v3-core/contracts/interfaces/IPoolAddressesProvider.sol";

contract FlashLoanExample is FlashLoanSimpleReceiverBase {

    constructor(IPoolAddressesProvider provider)
        FlashLoanSimpleReceiverBase(provider) {}

    // Функция, вызываемая Aave после получения flash loan
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {

        // 1. Получили amount токенов asset

        // 2. Выполняем арбитражную логику
        _executeArbitrage(asset, amount, params);

        // 3. Возвращаем кредит + комиссию
        uint256 amountOwed = amount + premium;
        IERC20(asset).approve(address(POOL), amountOwed);

        return true;
    }

    function requestFlashLoan(address asset, uint256 amount) external {
        POOL.flashLoanSimple(
            address(this),  // receiverAddress
            asset,          // asset
            amount,         // amount
            "",             // params
            0               // referralCode
        );
    }

    function _executeArbitrage(
        address asset,
        uint256 amount,
        bytes calldata params
    ) internal {
        // Здесь логика арбитража между DEX
    }
}
```

### Применение Flash Loans

1. **Арбитраж** — покупка на одной DEX, продажа на другой
2. **Ликвидация** — ликвидация позиций без собственного капитала
3. **Рефинансирование** — перенос залога между протоколами
4. **Self-liquidation** — закрытие своей позиции без продажи залога

## Риски DeFi

### 1. Smart Contract Risk
```solidity
// Пример уязвимости — reentrancy
contract VulnerableProtocol {
    mapping(address => uint256) public balances;

    // УЯЗВИМАЯ функция
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        balances[msg.sender] = 0; // Слишком поздно!
    }

    // БЕЗОПАСНАЯ версия с Checks-Effects-Interactions
    function withdrawSafe() external {
        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0; // Сначала меняем состояние
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
    }
}
```

### 2. Oracle Risk
Манипуляция ценами через оракулы.

### 3. Impermanent Loss
Потери от предоставления ликвидности на DEX (подробнее в разделе DEX).

### 4. Liquidation Risk
Ликвидация позиции при падении цены залога (подробнее в Lending).

### 5. Governance Risk
Атаки через governance токены.

## Безопасность в DeFi

### Проверки перед использованием протокола

1. **Аудиты** — проверить наличие аудитов от Trail of Bits, OpenZeppelin, Certik
2. **TVL и время работы** — чем дольше работает и больше TVL, тем надежнее
3. **Код** — проверен ли контракт на Etherscan
4. **Команда** — известна ли команда (хотя анонимность не всегда плохо)
5. **Страховка** — есть ли покрытие через Nexus Mutual

### Типичные атаки

```
┌─────────────────────────────────────────────────────────────┐
│                    DeFi Attack Vectors                      │
├─────────────────────────────────────────────────────────────┤
│  • Reentrancy attacks                                       │
│  • Flash loan attacks                                       │
│  • Oracle manipulation                                      │
│  • Governance attacks                                       │
│  • Front-running / MEV                                      │
│  • Infinite mint bugs                                       │
│  • Access control issues                                    │
└─────────────────────────────────────────────────────────────┘
```

## Интеграция с DeFi протоколами

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

interface IAaveLendingPool {
    function deposit(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external;
}

contract DeFiIntegration {
    IUniswapV2Router public uniswap;
    IAaveLendingPool public aave;

    constructor(address _uniswap, address _aave) {
        uniswap = IUniswapV2Router(_uniswap);
        aave = IAaveLendingPool(_aave);
    }

    // Swap токены и депозит в Aave
    function swapAndDeposit(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut
    ) external {
        // 1. Swap на Uniswap
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;

        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(uniswap), amountIn);

        uint[] memory amounts = uniswap.swapExactTokensForTokens(
            amountIn,
            minAmountOut,
            path,
            address(this),
            block.timestamp + 300
        );

        // 2. Депозит в Aave
        uint256 amountOut = amounts[1];
        IERC20(tokenOut).approve(address(aave), amountOut);
        aave.deposit(tokenOut, amountOut, msg.sender, 0);
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

## DeFi на разных блокчейнах

| Блокчейн | Особенности | Популярные протоколы |
|----------|-------------|---------------------|
| Ethereum | Основной, высокие газ-комиссии | Aave, Uniswap, Curve |
| Arbitrum | L2, низкие комиссии | GMX, Radiant |
| Polygon | Sidechain, очень дешево | QuickSwap, Aave |
| BSC | Централизованный, дешевый | PancakeSwap, Venus |
| Solana | Высокая скорость | Raydium, Marinade |
| Avalanche | Быстрый finality | Trader Joe, Benqi |

## Полезные ресурсы

- [DeFi Llama](https://defillama.com/) — аналитика TVL
- [Dune Analytics](https://dune.com/) — on-chain аналитика
- [DeBank](https://debank.com/) — портфолио трекер
- [Etherscan](https://etherscan.io/) — блокчейн эксплорер

## Заключение

DeFi представляет собой революцию в финансах, предлагая открытый доступ к финансовым услугам без посредников. Однако это сопряжено с рисками: смарт-контракты могут содержать уязвимости, оракулы могут быть манипулированы, а высокая волатильность криптовалют создает риск ликвидации.

Для успешной работы с DeFi необходимо:
- Понимать механизмы работы протоколов
- Оценивать риски перед использованием
- Диверсифицировать позиции
- Следить за здоровьем своих позиций (особенно в lending)
- Использовать только проверенные протоколы с аудитами
