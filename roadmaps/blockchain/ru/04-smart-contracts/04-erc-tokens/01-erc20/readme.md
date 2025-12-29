# ERC-20

## Введение

**ERC-20** — это стандарт токенов в сети Ethereum, определяющий набор правил для создания взаимозаменяемых токенов. Предложен в ноябре 2015 года Фабианом Фогельштеллером и стал самым используемым стандартом в криптоиндустрии.

## Что такое взаимозаменяемые токены?

Взаимозаменяемость означает, что каждая единица токена идентична другой:
- 1 USDT в вашем кошельке = 1 USDT в любом другом кошельке
- Токены делимы (можно отправить 0.001 токена)
- Нет уникальных идентификаторов у отдельных токенов

## Интерфейс ERC-20

### Обязательные функции

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    // Возвращает общее количество токенов
    function totalSupply() external view returns (uint256);

    // Возвращает баланс указанного адреса
    function balanceOf(address account) external view returns (uint256);

    // Переводит токены получателю
    function transfer(address to, uint256 amount) external returns (bool);

    // Возвращает количество токенов, которое spender может потратить от имени owner
    function allowance(address owner, address spender) external view returns (uint256);

    // Разрешает spender тратить указанное количество токенов
    function approve(address spender, uint256 amount) external returns (bool);

    // Переводит токены от одного адреса к другому (требует approve)
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

### Обязательные события

```solidity
// Вызывается при любом переводе токенов (включая минтинг и сжигание)
event Transfer(address indexed from, address indexed to, uint256 value);

// Вызывается при вызове approve
event Approval(address indexed owner, address indexed spender, uint256 value);
```

### Опциональные функции (ERC-20 Metadata)

```solidity
interface IERC20Metadata is IERC20 {
    // Название токена (например, "Tether USD")
    function name() external view returns (string memory);

    // Символ токена (например, "USDT")
    function symbol() external view returns (string memory);

    // Количество десятичных знаков (обычно 18)
    function decimals() external view returns (uint8);
}
```

## Полная реализация ERC-20

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyToken {
    // Метаданные токена
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;

    // Маппинги для хранения балансов и разрешений
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    // События
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = 18;

        // Минтим начальное количество токенов создателю
        _mint(msg.sender, _initialSupply * 10 ** decimals);
    }

    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) public returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function allowance(address owner, address spender) public view returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public returns (bool) {
        // Проверяем и уменьшаем allowance
        uint256 currentAllowance = _allowances[from][msg.sender];
        require(currentAllowance >= amount, "ERC20: insufficient allowance");
        unchecked {
            _approve(from, msg.sender, currentAllowance - amount);
        }

        _transfer(from, to, amount);
        return true;
    }

    // Внутренние функции
    function _transfer(
        address from,
        address to,
        uint256 amount
    ) internal {
        require(from != address(0), "ERC20: transfer from zero address");
        require(to != address(0), "ERC20: transfer to zero address");

        uint256 fromBalance = _balances[from];
        require(fromBalance >= amount, "ERC20: insufficient balance");

        unchecked {
            _balances[from] = fromBalance - amount;
            _balances[to] += amount;
        }

        emit Transfer(from, to, amount);
    }

    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal {
        require(owner != address(0), "ERC20: approve from zero address");
        require(spender != address(0), "ERC20: approve to zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: mint to zero address");

        totalSupply += amount;
        unchecked {
            _balances[account] += amount;
        }

        emit Transfer(address(0), account, amount);
    }

    function _burn(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: burn from zero address");

        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "ERC20: burn exceeds balance");

        unchecked {
            _balances[account] = accountBalance - amount;
            totalSupply -= amount;
        }

        emit Transfer(account, address(0), amount);
    }
}
```

## Использование OpenZeppelin

OpenZeppelin предоставляет проверенную и безопасную реализацию ERC-20:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, ERC20Burnable, ERC20Pausable, Ownable {
    constructor(address initialOwner)
        ERC20("MyToken", "MTK")
        Ownable(initialOwner)
    {
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }

    // Только владелец может минтить новые токены
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    // Только владелец может ставить на паузу
    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    // Переопределение для поддержки паузы
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Pausable)
    {
        super._update(from, to, value);
    }
}
```

## Популярные ERC-20 токены

### USDT (Tether)

- **Тип:** Стейблкоин, привязанный к USD
- **Decimals:** 6 (не 18!)
- **Особенности:** Централизованный, можно заморозить адреса

### USDC (USD Coin)

- **Тип:** Стейблкоин от Circle
- **Decimals:** 6
- **Особенности:** Регулируемый, прозрачные резервы

### LINK (Chainlink)

- **Тип:** Utility токен
- **Decimals:** 18
- **Использование:** Оплата услуг оракулов

## Механизм Approve и TransferFrom

Этот механизм необходим для взаимодействия с DeFi протоколами:

```
1. Пользователь вызывает approve(DEX_address, amount)
2. DEX теперь может переводить токены от имени пользователя
3. DEX вызывает transferFrom(user, DEX, amount) при свопе
```

```solidity
// Пример использования в коде
contract TokenSwap {
    IERC20 public tokenA;
    IERC20 public tokenB;

    function swap(uint256 amountA) external {
        // Требует предварительного approve от пользователя
        tokenA.transferFrom(msg.sender, address(this), amountA);

        // Рассчитываем и отправляем tokenB
        uint256 amountB = calculateSwap(amountA);
        tokenB.transfer(msg.sender, amountB);
    }
}
```

## Безопасность ERC-20

### Проблема Approve Race Condition

```solidity
// Опасный сценарий:
// 1. Алиса одобрила Бобу 100 токенов
// 2. Алиса хочет изменить на 50 токенов
// 3. Боб видит транзакцию и успевает потратить 100
// 4. После изменения approve Боб тратит ещё 50

// Решение: сначала установить в 0, потом в новое значение
function safeApprove(IERC20 token, address spender, uint256 amount) internal {
    token.approve(spender, 0);
    token.approve(spender, amount);
}
```

### Проверка возвращаемого значения

Некоторые токены не возвращают `bool` из `transfer`:

```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract SafeTokenHandler {
    using SafeERC20 for IERC20;

    function safeTransfer(IERC20 token, address to, uint256 amount) external {
        // Безопасно работает с любыми ERC-20 токенами
        token.safeTransfer(to, amount);
    }
}
```

## Расширения ERC-20

### ERC20Permit (EIP-2612)

Позволяет делать approve через подпись, без отдельной транзакции:

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract MyToken is ERC20, ERC20Permit {
    constructor() ERC20("MyToken", "MTK") ERC20Permit("MyToken") {}
}

// Использование
function permitAndTransfer(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external {
    token.permit(owner, spender, value, deadline, v, r, s);
    token.transferFrom(owner, address(this), value);
}
```

### ERC20Votes

Для governance токенов с механизмом голосования:

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";

contract GovernanceToken is ERC20, ERC20Permit, ERC20Votes {
    constructor() ERC20("GovToken", "GOV") ERC20Permit("GovToken") {}

    // Необходимые переопределения
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Votes)
    {
        super._update(from, to, value);
    }

    function nonces(address owner)
        public view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
}
```

## Типичные ошибки

1. **Забыть про decimals** — 1 токен = 1 * 10^18 wei
2. **Не проверять адрес 0** — приводит к потере токенов
3. **Игнорировать возвращаемое значение** — некоторые токены могут вернуть false
4. **Не использовать SafeERC20** — проблемы с нестандартными токенами

## Дополнительные ресурсы

- [EIP-20 Specification](https://eips.ethereum.org/EIPS/eip-20)
- [OpenZeppelin ERC20](https://docs.openzeppelin.com/contracts/5.x/erc20)
- [Token Best Practices](https://consensys.github.io/smart-contract-best-practices/tokens/)
