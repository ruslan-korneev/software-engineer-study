# Yield Farming

## Введение

**Yield Farming** (фермерство доходности) — это стратегия максимизации дохода в DeFi путем предоставления ликвидности или стейкинга токенов в различных протоколах. Пользователи получают вознаграждения в виде процентов, комиссий и токенов управления.

## Основные источники дохода

```
┌─────────────────────────────────────────────────────────────┐
│                   Yield Sources                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Trading Fees         Комиссии от свапов на DEX         │
│     └─► 0.3% Uniswap, 0.04% Curve                          │
│                                                             │
│  2. Interest             Проценты от lending               │
│     └─► Aave, Compound (1-10% APY)                         │
│                                                             │
│  3. Token Rewards        Награды токенами протокола        │
│     └─► COMP, AAVE, CRV, SUSHI                             │
│                                                             │
│  4. Staking Rewards      Стейкинг governance токенов       │
│     └─► veCRV, xSUSHI                                      │
│                                                             │
│  5. Arbitrage            Автоматический арбитраж           │
│     └─► Yearn vaults                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Liquidity Mining

### Концепция

Протоколы раздают свои токены пользователям, которые предоставляют ликвидность.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract LiquidityMining {
    using SafeERC20 for IERC20;

    IERC20 public lpToken;           // LP токен для стейкинга
    IERC20 public rewardToken;       // Токен награды

    uint256 public rewardPerBlock;   // Награда за блок
    uint256 public lastRewardBlock;  // Последний блок с наградами
    uint256 public accRewardPerShare; // Накопленная награда на единицу стейка

    uint256 public totalStaked;

    struct UserInfo {
        uint256 amount;      // Количество застейканных LP токенов
        uint256 rewardDebt;  // Уже учтенная награда
    }

    mapping(address => UserInfo) public userInfo;

    constructor(
        address _lpToken,
        address _rewardToken,
        uint256 _rewardPerBlock
    ) {
        lpToken = IERC20(_lpToken);
        rewardToken = IERC20(_rewardToken);
        rewardPerBlock = _rewardPerBlock;
        lastRewardBlock = block.number;
    }

    // Обновление накопленных наград
    function updatePool() public {
        if (block.number <= lastRewardBlock) {
            return;
        }

        if (totalStaked == 0) {
            lastRewardBlock = block.number;
            return;
        }

        uint256 blocks = block.number - lastRewardBlock;
        uint256 reward = blocks * rewardPerBlock;

        // Награда на единицу стейка (с точностью 1e12)
        accRewardPerShare += (reward * 1e12) / totalStaked;
        lastRewardBlock = block.number;
    }

    // Расчет pending награды
    function pendingReward(address _user) public view returns (uint256) {
        UserInfo storage user = userInfo[_user];

        uint256 _accRewardPerShare = accRewardPerShare;

        if (block.number > lastRewardBlock && totalStaked > 0) {
            uint256 blocks = block.number - lastRewardBlock;
            uint256 reward = blocks * rewardPerBlock;
            _accRewardPerShare += (reward * 1e12) / totalStaked;
        }

        return (user.amount * _accRewardPerShare / 1e12) - user.rewardDebt;
    }

    // Депозит LP токенов
    function deposit(uint256 _amount) external {
        UserInfo storage user = userInfo[msg.sender];

        updatePool();

        // Выплата pending награды
        if (user.amount > 0) {
            uint256 pending = (user.amount * accRewardPerShare / 1e12) - user.rewardDebt;
            if (pending > 0) {
                rewardToken.safeTransfer(msg.sender, pending);
            }
        }

        // Депозит
        if (_amount > 0) {
            lpToken.safeTransferFrom(msg.sender, address(this), _amount);
            user.amount += _amount;
            totalStaked += _amount;
        }

        user.rewardDebt = user.amount * accRewardPerShare / 1e12;
    }

    // Вывод LP токенов
    function withdraw(uint256 _amount) external {
        UserInfo storage user = userInfo[msg.sender];
        require(user.amount >= _amount, "Insufficient balance");

        updatePool();

        // Выплата pending награды
        uint256 pending = (user.amount * accRewardPerShare / 1e12) - user.rewardDebt;
        if (pending > 0) {
            rewardToken.safeTransfer(msg.sender, pending);
        }

        // Вывод
        if (_amount > 0) {
            user.amount -= _amount;
            totalStaked -= _amount;
            lpToken.safeTransfer(msg.sender, _amount);
        }

        user.rewardDebt = user.amount * accRewardPerShare / 1e12;
    }

    // Claim только наград
    function harvest() external {
        UserInfo storage user = userInfo[msg.sender];

        updatePool();

        uint256 pending = (user.amount * accRewardPerShare / 1e12) - user.rewardDebt;

        if (pending > 0) {
            rewardToken.safeTransfer(msg.sender, pending);
        }

        user.rewardDebt = user.amount * accRewardPerShare / 1e12;
    }
}
```

## Стратегии Yield Farming

### 1. Simple LP Farming

```solidity
contract SimpleFarmingStrategy {
    // 1. Предоставить ликвидность на Uniswap
    // 2. Застейкать LP токены в farming контракт
    // 3. Получать награды в токенах протокола
    // 4. Продавать награды или реинвестировать

    function execute() external {
        // Добавляем ликвидность
        uniswapRouter.addLiquidity(...);

        // Стейкаем LP токены
        farmingContract.deposit(lpAmount);

        // Периодически собираем награды
        // farmingContract.harvest();
    }
}
```

### 2. Leveraged Yield Farming

Использование заемных средств для увеличения позиции.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LeveragedFarming {
    // Пример: 3x leverage farming
    // 1. Депозит $1000 USDC
    // 2. Занять $2000 USDC из lending протокола
    // 3. Всего $3000 работает на yield

    struct Position {
        uint256 collateral;
        uint256 borrowed;
        uint256 lpTokens;
    }

    mapping(address => Position) public positions;

    function openLeveragedPosition(
        uint256 collateral,
        uint256 leverage  // в базисных пунктах (30000 = 3x)
    ) external {
        require(leverage <= 30000, "Max 3x leverage");

        // Получаем коллатерал от пользователя
        IERC20(usdc).transferFrom(msg.sender, address(this), collateral);

        // Занимаем дополнительно
        uint256 borrowAmount = (collateral * (leverage - 10000)) / 10000;
        lendingPool.borrow(usdc, borrowAmount);

        uint256 totalAmount = collateral + borrowAmount;

        // Добавляем ликвидность и стейкаем
        uint256 lpTokens = _addLiquidityAndStake(totalAmount);

        positions[msg.sender] = Position({
            collateral: collateral,
            borrowed: borrowAmount,
            lpTokens: lpTokens
        });
    }

    function _addLiquidityAndStake(uint256 amount) internal returns (uint256) {
        // Логика добавления ликвидности
        return 0;
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
}

interface ILendingPool {
    function borrow(address asset, uint256 amount) external;
}
```

### 3. Auto-compounding

Автоматическое реинвестирование наград.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract AutoCompoundVault {
    IERC20 public lpToken;
    IERC20 public rewardToken;
    IFarm public farm;
    IRouter public router;

    uint256 public totalShares;
    mapping(address => uint256) public shares;

    // Депозит с получением shares
    function deposit(uint256 _amount) external {
        lpToken.transferFrom(msg.sender, address(this), _amount);

        uint256 totalLP = lpToken.balanceOf(address(this)) + stakedLP();
        uint256 sharesToMint;

        if (totalShares == 0) {
            sharesToMint = _amount;
        } else {
            sharesToMint = (_amount * totalShares) / (totalLP - _amount);
        }

        shares[msg.sender] += sharesToMint;
        totalShares += sharesToMint;

        // Стейкаем все LP
        farm.deposit(_amount);
    }

    // Harvest и compound
    function harvest() public {
        // 1. Собираем награды
        farm.harvest();

        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        if (rewardBalance == 0) return;

        // 2. Свапаем награды в underlying токены
        uint256 half = rewardBalance / 2;
        _swap(address(rewardToken), address(token0), half);
        _swap(address(rewardToken), address(token1), rewardBalance - half);

        // 3. Добавляем ликвидность
        uint256 newLP = _addLiquidity();

        // 4. Стейкаем новые LP токены
        farm.deposit(newLP);

        // Shares не меняются, но стоимость каждой share растет
    }

    // Вывод
    function withdraw(uint256 _shares) external {
        require(shares[msg.sender] >= _shares, "Insufficient shares");

        uint256 totalLP = stakedLP();
        uint256 lpToWithdraw = (_shares * totalLP) / totalShares;

        shares[msg.sender] -= _shares;
        totalShares -= _shares;

        farm.withdraw(lpToWithdraw);
        lpToken.transfer(msg.sender, lpToWithdraw);
    }

    function stakedLP() public view returns (uint256) {
        return farm.userInfo(address(this)).amount;
    }

    function _swap(address from, address to, uint256 amount) internal {
        // Логика свапа
    }

    function _addLiquidity() internal returns (uint256) {
        // Логика добавления ликвидности
        return 0;
    }

    IERC20 public token0;
    IERC20 public token1;
}

interface IFarm {
    function deposit(uint256 amount) external;
    function withdraw(uint256 amount) external;
    function harvest() external;
    function userInfo(address user) external view returns (UserInfo memory);

    struct UserInfo {
        uint256 amount;
        uint256 rewardDebt;
    }
}

interface IRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

## Yearn Finance

Автоматизированные стратегии yield farming.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Упрощенный интерфейс Yearn Vault
interface IYearnVault {
    function deposit(uint256 _amount) external returns (uint256);
    function withdraw(uint256 _shares) external returns (uint256);
    function pricePerShare() external view returns (uint256);
    function token() external view returns (address);
}

contract YearnIntegration {
    function depositToVault(
        address vault,
        uint256 amount
    ) external returns (uint256 shares) {
        address token = IYearnVault(vault).token();

        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(vault, amount);

        shares = IYearnVault(vault).deposit(amount);

        // Возвращаем shares пользователю
        IERC20(vault).transfer(msg.sender, shares);
    }

    function withdrawFromVault(
        address vault,
        uint256 shares
    ) external returns (uint256 amount) {
        IERC20(vault).transferFrom(msg.sender, address(this), shares);

        amount = IYearnVault(vault).withdraw(shares);

        address token = IYearnVault(vault).token();
        IERC20(token).transfer(msg.sender, amount);
    }

    // Расчет текущей стоимости shares
    function getShareValue(
        address vault,
        uint256 shares
    ) external view returns (uint256) {
        uint256 pricePerShare = IYearnVault(vault).pricePerShare();
        return (shares * pricePerShare) / 1e18;
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

## Convex Finance (Curve Boosting)

Увеличение наград от Curve через boosting.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IConvexBooster {
    function deposit(uint256 _pid, uint256 _amount, bool _stake) external returns (bool);
    function withdraw(uint256 _pid, uint256 _amount) external returns (bool);
}

interface IConvexRewards {
    function stake(uint256 _amount) external returns (bool);
    function withdraw(uint256 _amount, bool claim) external returns (bool);
    function getReward() external returns (bool);
    function earned(address account) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}

contract ConvexStrategy {
    IConvexBooster public booster;
    IConvexRewards public rewardPool;
    IERC20 public curveLpToken;

    uint256 public pid; // Pool ID на Convex

    // Депозит Curve LP токенов через Convex
    function deposit(uint256 amount) external {
        curveLpToken.transferFrom(msg.sender, address(this), amount);
        curveLpToken.approve(address(booster), amount);

        // Депозит в Convex с автоматическим стейкингом
        booster.deposit(pid, amount, true);
    }

    // Claim наград (CRV + CVX + extras)
    function claimRewards() external {
        rewardPool.getReward();

        // Награды:
        // - CRV (boosted)
        // - CVX
        // - Возможно другие токены
    }

    function pendingRewards(address user) external view returns (uint256) {
        return rewardPool.earned(user);
    }
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

## Расчет APY

### Формулы

```solidity
library APYCalculation {
    // APR = (Rewards per year / Total staked) * 100

    function calculateAPR(
        uint256 rewardsPerSecond,
        uint256 rewardPrice,      // в USD, 1e18
        uint256 totalStaked,      // в токенах, 1e18
        uint256 stakedTokenPrice  // в USD, 1e18
    ) public pure returns (uint256) {
        uint256 rewardsPerYear = rewardsPerSecond * 365 days;
        uint256 rewardValuePerYear = (rewardsPerYear * rewardPrice) / 1e18;
        uint256 totalStakedValue = (totalStaked * stakedTokenPrice) / 1e18;

        // APR в базисных пунктах (10000 = 100%)
        return (rewardValuePerYear * 10000) / totalStakedValue;
    }

    // APY = (1 + APR/n)^n - 1
    // где n = количество компаундингов в год

    function aprToApy(
        uint256 apr,        // в базисных пунктах
        uint256 compounds   // количество компаундингов в год
    ) public pure returns (uint256) {
        // Упрощенная формула для малых APR:
        // APY ≈ APR + APR^2/(2*n)

        uint256 aprSquared = (apr * apr) / 10000;
        uint256 compound = aprSquared / (2 * compounds);

        return apr + compound;
    }
}
```

### Пример расчета

```
Pool: ETH-USDC на Uniswap
TVL: $10,000,000
Trading fees (0.3%): $50,000/день
Reward emissions: 1000 UNI/день = $5,000/день

Trading Fee APR:
= ($50,000 × 365) / $10,000,000 × 100
= 182.5%

Reward APR:
= ($5,000 × 365) / $10,000,000 × 100
= 18.25%

Total APR = 200.75%

APY (daily compounding):
= (1 + 2.0075/365)^365 - 1
= 642.5%
```

## Риски Yield Farming

### 1. Impermanent Loss

```
┌─────────────────────────────────────────────────────────────┐
│                   Impermanent Loss Risk                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Price Change    │    IL      │   Need APY to Break Even   │
│   ─────────────────────────────────────────────────────────│
│       ±25%        │   0.6%     │        0.6%                │
│       ±50%        │   2.0%     │        2.0%                │
│       ±100%       │   5.7%     │        5.7%                │
│       ±200%       │   13.4%    │        13.4%               │
│       ±500%       │   25.5%    │        25.5%               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. Smart Contract Risk

```solidity
// Проверка безопасности перед фармингом
contract SafetyChecks {
    function checkProtocol(address protocol) external view returns (bool safe) {
        // 1. Проверить наличие аудита
        // 2. Проверить TVL и историю
        // 3. Проверить ownership и admin keys
        // 4. Проверить код на Etherscan
        // 5. Проверить наличие timelock

        return true; // Упрощенно
    }
}
```

### 3. Token Risk

- Инфляция reward токенов
- Падение цены reward токенов
- Rug pulls

### 4. Ликвидационный риск

При leveraged farming.

## Инструменты для Yield Farming

### Aggregators

- **Yearn Finance** — автоматизированные vault стратегии
- **Beefy Finance** — auto-compounding вaults
- **Harvest Finance** — yield optimization

### Analytics

- **DeFi Llama** — TVL и APY по протоколам
- **DeBank** — портфолио трекинг
- **Zapper** — управление позициями
- **APY.vision** — анализ IL и доходности

## Пример полной стратегии

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract YieldFarmingStrategy {
    // Стратегия: Curve + Convex + Auto-compound

    ICurvePool public curvePool;
    IConvexBooster public convex;
    IConvexRewards public convexRewards;
    IUniswapRouter public router;

    IERC20 public dai;
    IERC20 public usdc;
    IERC20 public usdt;
    IERC20 public crv;
    IERC20 public cvx;

    // Депозит стейблкоинов в стратегию
    function deposit(
        uint256 daiAmount,
        uint256 usdcAmount,
        uint256 usdtAmount
    ) external {
        // 1. Получаем токены
        if (daiAmount > 0) dai.transferFrom(msg.sender, address(this), daiAmount);
        if (usdcAmount > 0) usdc.transferFrom(msg.sender, address(this), usdcAmount);
        if (usdtAmount > 0) usdt.transferFrom(msg.sender, address(this), usdtAmount);

        // 2. Добавляем ликвидность в Curve 3pool
        dai.approve(address(curvePool), daiAmount);
        usdc.approve(address(curvePool), usdcAmount);
        usdt.approve(address(curvePool), usdtAmount);

        uint256[3] memory amounts = [daiAmount, usdcAmount, usdtAmount];
        uint256 lpTokens = curvePool.add_liquidity(amounts, 0);

        // 3. Депозит LP в Convex для boosted rewards
        IERC20(curvePool).approve(address(convex), lpTokens);
        convex.deposit(0, lpTokens, true); // pid=0 для 3pool
    }

    // Harvest и реинвест
    function harvest() external {
        // 1. Claim CRV и CVX
        convexRewards.getReward();

        uint256 crvBalance = crv.balanceOf(address(this));
        uint256 cvxBalance = cvx.balanceOf(address(this));

        // 2. Swap rewards в стейблкоины
        if (crvBalance > 0) {
            _swapToStable(address(crv), crvBalance);
        }
        if (cvxBalance > 0) {
            _swapToStable(address(cvx), cvxBalance);
        }

        // 3. Добавляем ликвидность и реинвестируем
        uint256 usdcBalance = usdc.balanceOf(address(this));
        if (usdcBalance > 0) {
            usdc.approve(address(curvePool), usdcBalance);
            uint256[3] memory amounts = [uint256(0), usdcBalance, uint256(0)];
            uint256 newLP = curvePool.add_liquidity(amounts, 0);

            IERC20(curvePool).approve(address(convex), newLP);
            convex.deposit(0, newLP, true);
        }
    }

    function _swapToStable(address token, uint256 amount) internal {
        IERC20(token).approve(address(router), amount);

        address[] memory path = new address[](2);
        path[0] = token;
        path[1] = address(usdc);

        router.swapExactTokensForTokens(
            amount,
            0,
            path,
            address(this),
            block.timestamp
        );
    }
}

interface ICurvePool {
    function add_liquidity(uint256[3] memory amounts, uint256 min_mint) external returns (uint256);
}

interface IConvexBooster {
    function deposit(uint256 pid, uint256 amount, bool stake) external returns (bool);
}

interface IConvexRewards {
    function getReward() external returns (bool);
}

interface IUniswapRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

## Best Practices

### 1. Диверсификация

```
Пример портфеля:
- 40% — Stablecoin farming (Curve 3pool)
- 30% — Blue-chip LP (ETH-USDC на Uniswap)
- 20% — Single-sided staking (ETH в Lido)
- 10% — High-risk/high-reward farms
```

### 2. Управление рисками

- Не инвестировать больше, чем готовы потерять
- Проверять аудиты протоколов
- Мониторить позиции ежедневно
- Иметь план выхода

### 3. Учет затрат

```
Затраты на farming:
- Gas fees на депозит/вывод
- Gas fees на harvest
- Impermanent Loss
- Налоги на прибыль

ROI = (Rewards - IL - Gas - Taxes) / Initial Investment
```

### 4. Автоматизация

- Использовать auto-compounding vaults
- Настроить alerts на Health Factor
- Использовать Gelato/Chainlink Keepers для автоматизации

## Полезные ресурсы

- [DeFi Llama Yields](https://defillama.com/yields) — сравнение APY
- [Zapper](https://zapper.fi/) — управление позициями
- [DeBank](https://debank.com/) — портфолио трекер
- [APY.vision](https://apy.vision/) — анализ IL
- [Yearn Docs](https://docs.yearn.fi/) — документация Yearn
