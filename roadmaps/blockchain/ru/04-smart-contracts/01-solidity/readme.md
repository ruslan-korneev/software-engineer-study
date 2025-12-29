# Solidity

## Введение

**Solidity** — это объектно-ориентированный язык программирования высокого уровня, разработанный специально для создания смарт-контрактов на платформе Ethereum и других EVM-совместимых блокчейнах.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Экосистема Solidity                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    Solidity Code (.sol)                                        │
│           │                                                     │
│           ▼                                                     │
│    ┌─────────────┐                                             │
│    │   Compiler  │  (solc)                                     │
│    │   (solc)    │                                             │
│    └──────┬──────┘                                             │
│           │                                                     │
│           ▼                                                     │
│    ┌─────────────┐     ┌─────────────┐                         │
│    │  Bytecode   │     │    ABI      │                         │
│    │  (binary)   │     │   (JSON)    │                         │
│    └──────┬──────┘     └──────┬──────┘                         │
│           │                   │                                 │
│           ▼                   ▼                                 │
│    ┌─────────────────────────────────┐                         │
│    │         EVM (Ethereum          │                         │
│    │      Virtual Machine)           │                         │
│    └─────────────────────────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## История создания

### Хронология развития

| Год | Событие |
|-----|---------|
| 2014 | Gavin Wood предложил концепцию Solidity |
| 2015 | Первый релиз вместе с запуском Ethereum |
| 2017 | Версия 0.4.x — массовое принятие (ICO бум) |
| 2019 | Версия 0.5.x — улучшения безопасности |
| 2020 | Версия 0.6.x — try/catch, виртуальные функции |
| 2021 | Версия 0.8.x — встроенная защита от overflow |
| 2023 | Версия 0.8.20+ — поддержка Shanghai/Cancun |
| 2024 | Активная разработка 0.8.24+ |

### Создатели

- **Gavin Wood** — автор идеи и Yellow Paper Ethereum
- **Christian Reitwiessner** — ведущий разработчик компилятора
- **Alex Beregszaszi** — соавтор, разработчик Yul

## Особенности языка

### Ключевые характеристики

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Пример базового контракта
 * @notice Демонстрирует основные особенности Solidity
 */
contract SolidityFeatures {
    // 1. Статическая типизация
    uint256 public counter;
    address public owner;

    // 2. Модификаторы доступа
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    // 3. События для логирования
    event CounterIncremented(address indexed by, uint256 newValue);

    // 4. Конструктор
    constructor() {
        owner = msg.sender;
    }

    // 5. Функции с различными спецификаторами
    function increment() public onlyOwner {
        counter++;
        emit CounterIncremented(msg.sender, counter);
    }

    // 6. View функции (только чтение)
    function getCounter() public view returns (uint256) {
        return counter;
    }

    // 7. Pure функции (без доступа к storage)
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
}
```

### Сравнение с другими языками

```
┌────────────────┬────────────────┬─────────────────┬────────────────┐
│   Критерий     │    Solidity    │    JavaScript   │     Rust       │
├────────────────┼────────────────┼─────────────────┼────────────────┤
│ Типизация      │ Статическая    │ Динамическая    │ Статическая    │
│ Парадигма      │ ООП + Contract │ Мульти          │ Мульти         │
│ Среда          │ EVM            │ V8/Browser      │ Native         │
│ Gas оптимиз.   │ Критично       │ Не применимо    │ Не применимо   │
│ Determinism    │ 100%           │ Нет             │ Зависит        │
│ Наследование   │ Множественное  │ Прототипное     │ Traits         │
└────────────────┴────────────────┴─────────────────┴────────────────┘
```

## Версионирование

### Директива pragma

```solidity
// Точная версия
pragma solidity 0.8.20;

// Любая версия 0.8.x
pragma solidity ^0.8.0;

// Диапазон версий
pragma solidity >=0.8.0 <0.9.0;

// Несколько условий
pragma solidity >=0.8.0 <=0.8.20;
```

### Основные изменения по версиям

#### Версия 0.8.x (рекомендуемая)

```solidity
// Встроенная защита от overflow/underflow
uint8 x = 255;
// x + 1 вызовет revert, а не переполнение

// Явные преобразования типов
uint256 big = 1000;
uint8 small = uint8(big); // Требуется явное преобразование

// Custom errors (экономия gas)
error InsufficientBalance(uint256 available, uint256 required);

function withdraw(uint256 amount) public {
    if (balance < amount)
        revert InsufficientBalance(balance, amount);
}
```

#### Важные версии 0.8.x

| Версия | Ключевые изменения |
|--------|-------------------|
| 0.8.0 | Overflow protection, ABIEncoderV2 по умолчанию |
| 0.8.4 | Custom errors |
| 0.8.8 | Оптимизация inline assembly |
| 0.8.13 | Using for на уровне файла |
| 0.8.18 | Отключение PUSH0 для совместимости |
| 0.8.20 | Поддержка Shanghai (PUSH0 по умолчанию) |
| 0.8.24 | Поддержка Cancun (transient storage) |

## Области применения

### DeFi (Децентрализованные финансы)

```solidity
// Пример простого пула ликвидности
contract SimpleLiquidityPool {
    IERC20 public tokenA;
    IERC20 public tokenB;

    function swap(address tokenIn, uint256 amountIn) external {
        // Логика обмена токенов
    }

    function addLiquidity(uint256 amountA, uint256 amountB) external {
        // Добавление ликвидности
    }
}
```

### NFT (Non-Fungible Tokens)

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract MyNFT is ERC721 {
    uint256 private _tokenIdCounter;

    constructor() ERC721("MyNFT", "MNFT") {}

    function mint(address to) public {
        _safeMint(to, _tokenIdCounter);
        _tokenIdCounter++;
    }
}
```

### DAO (Децентрализованные автономные организации)

```solidity
contract SimpleDAO {
    struct Proposal {
        string description;
        uint256 voteCount;
        bool executed;
    }

    mapping(uint256 => Proposal) public proposals;
    mapping(address => uint256) public votingPower;

    function vote(uint256 proposalId) external {
        proposals[proposalId].voteCount += votingPower[msg.sender];
    }
}
```

## Инструменты разработки

### Фреймворки

```
┌─────────────────────────────────────────────────────────────────┐
│                    Инструменты разработки                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Hardhat   │  │   Foundry   │  │   Truffle   │             │
│  │ (JavaScript)│  │   (Rust)    │  │ (JavaScript)│             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│        │               │                │                       │
│        └───────────────┼────────────────┘                       │
│                        ▼                                        │
│              ┌─────────────────┐                               │
│              │  Solidity Code  │                               │
│              └─────────────────┘                               │
│                                                                 │
│  IDE: VS Code + Solidity Extension, Remix IDE                  │
│  Testing: Mocha, Forge, Waffle                                 │
│  Security: Slither, Mythril, Echidna                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Установка компилятора

```bash
# Через npm (для Hardhat/Truffle)
npm install solc

# Через Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Напрямую (solc-select для управления версиями)
pip install solc-select
solc-select install 0.8.20
solc-select use 0.8.20
```

## Best Practices

### Рекомендации по написанию кода

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 1. Используйте именованные импорты
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// 2. Документируйте код с NatSpec
/// @title Безопасный токен
/// @author Ваше имя
/// @notice Токен с защитой от reentrancy
contract SafeToken is ERC20 {
    /// @notice Владелец контракта
    address public owner;

    /// @dev Защита от reentrancy
    bool private _locked;

    modifier nonReentrant() {
        require(!_locked, "Reentrancy");
        _locked = true;
        _;
        _locked = false;
    }

    constructor() ERC20("Safe", "SAFE") {
        owner = msg.sender;
    }
}
```

### Типичные ошибки

| Ошибка | Проблема | Решение |
|--------|----------|---------|
| Reentrancy | Повторный вход в функцию | Использовать nonReentrant |
| Integer overflow | Переполнение (до 0.8.0) | Использовать 0.8.x или SafeMath |
| tx.origin | Уязвимость авторизации | Использовать msg.sender |
| Unbounded loops | Превышение gas limit | Ограничить итерации |
| Front-running | Опережение транзакций | Commit-reveal схемы |

## Ресурсы для изучения

- [Официальная документация](https://docs.soliditylang.org/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Solidity by Example](https://solidity-by-example.org/)
- [CryptoZombies](https://cryptozombies.io/) — интерактивный курс
- [Ethernaut](https://ethernaut.openzeppelin.com/) — задачи по безопасности

## Заключение

Solidity является стандартом де-факто для разработки смарт-контрактов в экосистеме Ethereum. Язык активно развивается, добавляя новые возможности и улучшая безопасность. Понимание особенностей Solidity, включая gas-оптимизацию и паттерны безопасности, критически важно для создания надёжных децентрализованных приложений.

---

**Навигация:**
- [Основы Solidity](./01-basics/readme.md)
- [Продвинутый Solidity](./02-advanced/readme.md)
- [Паттерны проектирования](./03-patterns/readme.md)
