# Децентрализованные биржи (DEX)

## Введение

**DEX (Decentralized Exchange)** — это децентрализованная биржа, позволяющая обменивать криптовалюты напрямую между пользователями без посредников. В отличие от централизованных бирж (CEX), DEX работает полностью на смарт-контрактах.

## Типы DEX

### 1. Order Book DEX
Классическая модель с книгой ордеров (dYdX, Serum).

### 2. AMM (Automated Market Maker)
Автоматический маркет-мейкер с пулами ликвидности (Uniswap, Curve).

```
┌─────────────────────────────────────────────────────────────┐
│                    DEX Types Comparison                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Order Book DEX              AMM DEX                       │
│   ┌─────────────┐            ┌─────────────┐               │
│   │  Bid  │ Ask │            │   Pool      │               │
│   │ ───── │───── │           │ ┌───┬───┐   │               │
│   │ $1999 │$2001 │            │ │ETH│DAI│   │               │
│   │ $1998 │$2002 │            │ └───┴───┘   │               │
│   └─────────────┘            └─────────────┘               │
│                                                             │
│   + Лучшие цены для           + Всегда есть ликвидность    │
│     крупных ордеров           + Простота LP                │
│   - Требует ликвидность       - Impermanent Loss           │
│   - Сложнее on-chain          - Проскальзывание            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## AMM: Формула x * y = k

### Константа произведения (Constant Product)

Основная формула Uniswap V2:

```
x * y = k

где:
x = количество токена A в пуле
y = количество токена B в пуле
k = константа (инвариант)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleAMM {
    uint256 public reserveA;
    uint256 public reserveB;
    uint256 public k; // инвариант

    // Инициализация пула
    function initialize(uint256 amountA, uint256 amountB) external {
        reserveA = amountA;
        reserveB = amountB;
        k = amountA * amountB;
    }

    // Расчет выхода при свапе A -> B
    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256) {
        // Новый reserveIn после добавления amountIn
        uint256 newReserveIn = reserveIn + amountIn;

        // Из формулы x * y = k:
        // newReserveIn * newReserveOut = k
        // newReserveOut = k / newReserveIn
        uint256 newReserveOut = (reserveIn * reserveOut) / newReserveIn;

        // Количество на выход = текущий резерв - новый резерв
        return reserveOut - newReserveOut;
    }

    // Swap токена A на токен B
    function swapAforB(uint256 amountAIn) external returns (uint256 amountBOut) {
        amountBOut = getAmountOut(amountAIn, reserveA, reserveB);

        // Обновляем резервы
        reserveA += amountAIn;
        reserveB -= amountBOut;

        // Проверяем инвариант (с учетом погрешности)
        require(reserveA * reserveB >= k, "Invariant broken");

        // В реальности здесь transferFrom и transfer
        return amountBOut;
    }

    // Текущая цена A в терминах B
    function priceAinB() external view returns (uint256) {
        return (reserveB * 1e18) / reserveA;
    }
}
```

### Пример расчета

```
Начальное состояние пула:
- 100 ETH
- 200,000 DAI
- k = 100 * 200,000 = 20,000,000
- Цена: 1 ETH = 2,000 DAI

Swap: пользователь хочет купить ETH за 10,000 DAI

После добавления DAI:
- DAI в пуле: 200,000 + 10,000 = 210,000
- ETH в пуле: k / 210,000 = 20,000,000 / 210,000 = 95.238 ETH
- Пользователь получает: 100 - 95.238 = 4.762 ETH
- Эффективная цена: 10,000 / 4.762 = 2,100 DAI/ETH (хуже рыночной!)

Это называется Price Impact (влияние на цену)
```

### Price Impact (Проскальзывание)

```solidity
contract PriceImpact {
    function calculatePriceImpact(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256) {
        // Спот-цена до свапа
        uint256 spotPrice = (reserveOut * 1e18) / reserveIn;

        // Реальная цена при свапе
        uint256 amountOut = getAmountOut(amountIn, reserveIn, reserveOut);
        uint256 executionPrice = (amountOut * 1e18) / amountIn;

        // Price Impact = (spotPrice - executionPrice) / spotPrice * 100%
        return ((spotPrice - executionPrice) * 10000) / spotPrice;
    }

    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256) {
        uint256 newReserveIn = reserveIn + amountIn;
        uint256 newReserveOut = (reserveIn * reserveOut) / newReserveIn;
        return reserveOut - newReserveOut;
    }
}
```

## Uniswap

### Uniswap V2

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IUniswapV2Router02 {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function swapExactETHForTokens(
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external payable returns (uint[] memory amounts);

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

    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB);

    function getAmountsOut(
        uint amountIn,
        address[] calldata path
    ) external view returns (uint[] memory amounts);
}

contract UniswapV2Integration {
    IUniswapV2Router02 public immutable router;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    constructor(address _router) {
        router = IUniswapV2Router02(_router);
    }

    // Swap токенов
    function swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut
    ) external returns (uint256 amountOut) {
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(router), amountIn);

        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;

        uint[] memory amounts = router.swapExactTokensForTokens(
            amountIn,
            minAmountOut,
            path,
            msg.sender,
            block.timestamp + 300 // 5 минут deadline
        );

        return amounts[1];
    }

    // Multi-hop swap (например ETH -> USDC -> DAI)
    function multiHopSwap(
        address[] calldata path,
        uint256 amountIn,
        uint256 minAmountOut
    ) external returns (uint256) {
        IERC20(path[0]).transferFrom(msg.sender, address(this), amountIn);
        IERC20(path[0]).approve(address(router), amountIn);

        uint[] memory amounts = router.swapExactTokensForTokens(
            amountIn,
            minAmountOut,
            path,
            msg.sender,
            block.timestamp + 300
        );

        return amounts[amounts.length - 1];
    }

    // Добавление ликвидности
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountA,
        uint256 amountB
    ) external returns (uint256 liquidity) {
        IERC20(tokenA).transferFrom(msg.sender, address(this), amountA);
        IERC20(tokenB).transferFrom(msg.sender, address(this), amountB);

        IERC20(tokenA).approve(address(router), amountA);
        IERC20(tokenB).approve(address(router), amountB);

        (,, liquidity) = router.addLiquidity(
            tokenA,
            tokenB,
            amountA,
            amountB,
            (amountA * 95) / 100, // 5% slippage tolerance
            (amountB * 95) / 100,
            msg.sender,
            block.timestamp + 300
        );
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

### Uniswap V3: Concentrated Liquidity

В V3 LP выбирает ценовой диапазон для своей ликвидности.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IUniswapV3Pool {
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1);
}

interface INonfungiblePositionManager {
    struct MintParams {
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;
        int24 tickUpper;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        address recipient;
        uint256 deadline;
    }

    function mint(MintParams calldata params) external payable returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    );
}

contract UniswapV3Integration {
    INonfungiblePositionManager public positionManager;

    constructor(address _positionManager) {
        positionManager = INonfungiblePositionManager(_positionManager);
    }

    // Создание позиции с концентрированной ликвидностью
    function createPosition(
        address token0,
        address token1,
        uint24 fee,           // 500 = 0.05%, 3000 = 0.3%, 10000 = 1%
        int24 tickLower,      // нижняя граница цены
        int24 tickUpper,      // верхняя граница цены
        uint256 amount0,
        uint256 amount1
    ) external returns (uint256 tokenId) {
        IERC20(token0).transferFrom(msg.sender, address(this), amount0);
        IERC20(token1).transferFrom(msg.sender, address(this), amount1);

        IERC20(token0).approve(address(positionManager), amount0);
        IERC20(token1).approve(address(positionManager), amount1);

        INonfungiblePositionManager.MintParams memory params =
            INonfungiblePositionManager.MintParams({
                token0: token0,
                token1: token1,
                fee: fee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: amount0,
                amount1Desired: amount1,
                amount0Min: 0,
                amount1Min: 0,
                recipient: msg.sender,
                deadline: block.timestamp + 300
            });

        (tokenId,,,) = positionManager.mint(params);
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

## Impermanent Loss (Непостоянные потери)

### Что это такое?

Impermanent Loss — потери LP по сравнению с простым холдингом токенов.

```
Пример:
Начало: депозит 1 ETH ($2000) + 2000 DAI = $4000 total
Пул: 50% ETH / 50% DAI

ETH растет до $4000:
- Арбитражеры покупают ETH из пула
- В пуле теперь: 0.707 ETH + 2828 DAI
- Стоимость: 0.707 * $4000 + $2828 = $5656

Если бы просто холдили:
- 1 ETH * $4000 + $2000 = $6000

Impermanent Loss = $6000 - $5656 = $344 (5.7%)
```

### Формула Impermanent Loss

```solidity
library ImpermanentLoss {
    // IL = 2 * sqrt(priceRatio) / (1 + priceRatio) - 1
    // priceRatio = newPrice / oldPrice

    function calculateIL(
        uint256 initialPrice,
        uint256 currentPrice
    ) public pure returns (uint256) {
        // Возвращает IL в базисных пунктах (10000 = 100%)

        uint256 priceRatio = (currentPrice * 1e18) / initialPrice;

        // sqrt через Babylonian method
        uint256 sqrtRatio = sqrt(priceRatio);

        // 2 * sqrt(ratio) / (1 + ratio)
        uint256 numerator = 2 * sqrtRatio;
        uint256 denominator = 1e18 + priceRatio;

        uint256 lpValue = (numerator * 1e18) / denominator;

        // IL = 1 - lpValue
        if (lpValue >= 1e18) return 0;
        return 1e18 - lpValue;
    }

    function sqrt(uint256 x) internal pure returns (uint256) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        return y;
    }
}
```

### Таблица Impermanent Loss

| Изменение цены | IL |
|----------------|-----|
| 1.25x (+25%) | 0.6% |
| 1.50x (+50%) | 2.0% |
| 2x (+100%) | 5.7% |
| 3x (+200%) | 13.4% |
| 4x (+300%) | 20.0% |
| 5x (+400%) | 25.5% |

## Curve Finance: StableSwap

Оптимизирован для свапов стейблкоинов с минимальным проскальзыванием.

```solidity
// Curve использует модифицированную формулу:
// A * n^n * sum(x_i) + D = A * D * n^n + D^(n+1) / (n^n * prod(x_i))
// где A — параметр амплификации

interface ICurvePool {
    function exchange(
        int128 i,      // индекс входного токена
        int128 j,      // индекс выходного токена
        uint256 dx,    // количество входного
        uint256 min_dy // минимум выходного
    ) external returns (uint256);

    function get_dy(
        int128 i,
        int128 j,
        uint256 dx
    ) external view returns (uint256);

    function add_liquidity(
        uint256[3] calldata amounts,
        uint256 min_mint_amount
    ) external returns (uint256);
}

contract CurveIntegration {
    ICurvePool public pool; // например, 3pool (DAI/USDC/USDT)

    // Индексы токенов в 3pool:
    // 0 = DAI, 1 = USDC, 2 = USDT

    function swapStables(
        int128 fromToken,
        int128 toToken,
        uint256 amount,
        uint256 minReceived
    ) external returns (uint256) {
        // Curve имеет очень низкое проскальзывание для стейблов
        return pool.exchange(fromToken, toToken, amount, minReceived);
    }

    function getExpectedOutput(
        int128 from,
        int128 to,
        uint256 amount
    ) external view returns (uint256) {
        return pool.get_dy(from, to, amount);
    }
}
```

## DEX Aggregators

Агрегаторы находят лучший путь через несколько DEX.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface I1inchRouter {
    function swap(
        address executor,
        SwapDescription calldata desc,
        bytes calldata permit,
        bytes calldata data
    ) external payable returns (uint256 returnAmount);

    struct SwapDescription {
        address srcToken;
        address dstToken;
        address srcReceiver;
        address dstReceiver;
        uint256 amount;
        uint256 minReturnAmount;
        uint256 flags;
    }
}

contract DEXAggregatorExample {
    // Простой агрегатор для сравнения цен

    IUniswapV2Router public uniswap;
    ICurvePool public curve;

    struct Quote {
        uint256 amountOut;
        string dex;
    }

    function getBestQuote(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external view returns (Quote memory best) {
        // Проверяем Uniswap
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;

        try uniswap.getAmountsOut(amountIn, path) returns (uint[] memory amounts) {
            if (amounts[1] > best.amountOut) {
                best.amountOut = amounts[1];
                best.dex = "Uniswap";
            }
        } catch {}

        // Проверяем Curve (если это стейблы)
        // ...

        return best;
    }
}

interface IUniswapV2Router {
    function getAmountsOut(uint amountIn, address[] calldata path)
        external view returns (uint[] memory amounts);
}
```

## MEV и Front-running

### Проблема

Майнеры и боты могут видеть pending транзакции и:
1. **Front-running** — вставить свою транзакцию перед вашей
2. **Sandwich attack** — окружить вашу транзакцию своими
3. **Back-running** — выполнить арбитраж после вашей транзакции

```
Sandwich Attack:
1. Бот видит вашу транзакцию на покупку ETH за 10,000 USDC
2. Бот покупает ETH ДО вас (front-run)
3. Цена ETH растет
4. Ваша транзакция выполняется по худшей цене
5. Бот продает ETH ПОСЛЕ вас (back-run)
6. Бот получает прибыль, вы — убыток
```

### Защита от MEV

```solidity
contract MEVProtection {
    // 1. Использовать private mempool (Flashbots)
    // 2. Устанавливать жесткий slippage

    function safeSwap(
        address router,
        address[] calldata path,
        uint256 amountIn,
        uint256 expectedOut,
        uint256 maxSlippageBps  // базисные пункты (100 = 1%)
    ) external {
        uint256 minAmountOut = expectedOut * (10000 - maxSlippageBps) / 10000;

        IUniswapV2Router(router).swapExactTokensForTokens(
            amountIn,
            minAmountOut,  // защита от проскальзывания
            path,
            msg.sender,
            block.timestamp  // минимальный deadline
        );
    }
}
```

## Сравнение DEX

| DEX | Тип | Особенность | Сети |
|-----|-----|-------------|------|
| Uniswap V2 | AMM (x*y=k) | Простота | ETH, L2 |
| Uniswap V3 | Concentrated | Эффективность капитала | ETH, L2 |
| Curve | StableSwap | Низкое проскальзывание | Multi-chain |
| Balancer | Weighted Pools | Любые веса токенов | ETH, L2 |
| SushiSwap | AMM | Yield farming | Multi-chain |
| dYdX | Order Book | Деривативы, L2 | StarkEx |
| GMX | Perps | Perpetuals без IL | Arbitrum |

## Best Practices для работы с DEX

1. **Всегда устанавливайте slippage tolerance** — обычно 0.5-1%
2. **Проверяйте Price Impact** — избегайте свапов с >3% impact
3. **Используйте агрегаторы** — 1inch, Paraswap для лучших цен
4. **Учитывайте газ** — иногда лучше подождать низкий газ
5. **Остерегайтесь MEV** — используйте Flashbots Protect
6. **Проверяйте контракты токенов** — остерегайтесь honeypots

## Создание собственного AMM

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract SimpleAMM is ERC20 {
    IERC20 public immutable token0;
    IERC20 public immutable token1;

    uint256 public reserve0;
    uint256 public reserve1;

    uint256 public constant FEE = 3; // 0.3% fee

    constructor(address _token0, address _token1) ERC20("LP Token", "LP") {
        token0 = IERC20(_token0);
        token1 = IERC20(_token1);
    }

    function addLiquidity(
        uint256 amount0,
        uint256 amount1
    ) external returns (uint256 liquidity) {
        token0.transferFrom(msg.sender, address(this), amount0);
        token1.transferFrom(msg.sender, address(this), amount1);

        if (totalSupply() == 0) {
            liquidity = sqrt(amount0 * amount1);
        } else {
            liquidity = min(
                (amount0 * totalSupply()) / reserve0,
                (amount1 * totalSupply()) / reserve1
            );
        }

        require(liquidity > 0, "Insufficient liquidity minted");
        _mint(msg.sender, liquidity);

        reserve0 += amount0;
        reserve1 += amount1;
    }

    function removeLiquidity(uint256 liquidity) external returns (
        uint256 amount0,
        uint256 amount1
    ) {
        amount0 = (liquidity * reserve0) / totalSupply();
        amount1 = (liquidity * reserve1) / totalSupply();

        require(amount0 > 0 && amount1 > 0, "Insufficient liquidity burned");

        _burn(msg.sender, liquidity);

        reserve0 -= amount0;
        reserve1 -= amount1;

        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }

    function swap(
        address tokenIn,
        uint256 amountIn
    ) external returns (uint256 amountOut) {
        require(tokenIn == address(token0) || tokenIn == address(token1), "Invalid token");

        bool isToken0 = tokenIn == address(token0);

        (IERC20 tokenInContract, IERC20 tokenOutContract,
         uint256 reserveIn, uint256 reserveOut) = isToken0
            ? (token0, token1, reserve0, reserve1)
            : (token1, token0, reserve1, reserve0);

        tokenInContract.transferFrom(msg.sender, address(this), amountIn);

        // Применяем комиссию
        uint256 amountInWithFee = amountIn * (1000 - FEE);

        // x * y = k
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);

        tokenOutContract.transfer(msg.sender, amountOut);

        // Обновляем резервы
        if (isToken0) {
            reserve0 += amountIn;
            reserve1 -= amountOut;
        } else {
            reserve1 += amountIn;
            reserve0 -= amountOut;
        }
    }

    function sqrt(uint256 x) internal pure returns (uint256) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        return y;
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
}
```

## Полезные ресурсы

- [Uniswap Docs](https://docs.uniswap.org/)
- [Curve Resources](https://resources.curve.fi/)
- [DeFi Llama DEX](https://defillama.com/dexs)
- [DEX Screener](https://dexscreener.com/) — аналитика пулов
