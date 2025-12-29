# Продвинутый Solidity

## Введение

Продвинутые концепции Solidity включают наследование, интерфейсы, абстрактные контракты, библиотеки, события, обработку ошибок и низкоуровневое программирование с использованием assembly. Эти инструменты позволяют создавать сложные, модульные и эффективные смарт-контракты.

## Наследование

### Основы наследования

```
┌─────────────────────────────────────────────────────────────────┐
│                    Иерархия наследования                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────┐                              │
│                    │   Ownable   │  (базовый контракт)          │
│                    └──────┬──────┘                              │
│                           │                                     │
│              ┌────────────┼────────────┐                        │
│              │            │            │                        │
│              ▼            ▼            ▼                        │
│       ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│       │ Pausable │ │  ERC20   │ │AccessCtrl│                   │
│       └────┬─────┘ └────┬─────┘ └────┬─────┘                   │
│            │            │            │                          │
│            └────────────┼────────────┘                          │
│                         │                                       │
│                         ▼                                       │
│                  ┌─────────────┐                                │
│                  │  MyToken    │  (финальный контракт)          │
│                  └─────────────┘                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Базовый контракт
contract Ownable {
    address public owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Ownable: caller is not the owner");
        _;
    }

    // virtual позволяет переопределять функцию
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}

// Наследование
contract MyContract is Ownable {
    uint256 public value;

    // override переопределяет виртуальную функцию
    function transferOwnership(address newOwner) public override onlyOwner {
        require(newOwner != msg.sender, "Cannot transfer to self");
        super.transferOwnership(newOwner); // Вызов родительской функции
    }

    function setValue(uint256 _value) public onlyOwner {
        value = _value;
    }
}
```

### Множественное наследование

```solidity
contract A {
    uint256 public valueA;

    function foo() public virtual returns (string memory) {
        return "A";
    }
}

contract B {
    uint256 public valueB;

    function foo() public virtual returns (string memory) {
        return "B";
    }
}

// Множественное наследование (C3 linearization)
// Порядок важен: от более базового к более производному
contract C is A, B {
    // Должны указать все родительские контракты
    function foo() public override(A, B) returns (string memory) {
        return "C";
    }

    function callParents() public view returns (string memory, string memory) {
        return (A.foo(), B.foo());
    }
}

// Diamond problem решается через super и линеаризацию
contract D is A, B {
    function foo() public override(A, B) returns (string memory) {
        // super вызовет B.foo() (последний в списке наследования)
        return super.foo();
    }
}
```

### Конструкторы при наследовании

```solidity
contract Base1 {
    uint256 public x;

    constructor(uint256 _x) {
        x = _x;
    }
}

contract Base2 {
    uint256 public y;

    constructor(uint256 _y) {
        y = _y;
    }
}

// Способ 1: Аргументы в списке наследования
contract Derived1 is Base1(10), Base2(20) {
    // x = 10, y = 20
}

// Способ 2: Аргументы в конструкторе
contract Derived2 is Base1, Base2 {
    constructor(uint256 _x, uint256 _y) Base1(_x) Base2(_y) {
        // Дополнительная инициализация
    }
}

// Комбинированный подход
contract Derived3 is Base1(100), Base2 {
    constructor(uint256 _y) Base2(_y) {
        // x = 100, y = _y
    }
}
```

## Интерфейсы

```solidity
// Определение интерфейса
interface IERC20 {
    // Только сигнатуры функций
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    // События
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// Реализация интерфейса
contract MyToken is IERC20 {
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    uint256 private _totalSupply;

    constructor(uint256 initialSupply) {
        _mint(msg.sender, initialSupply);
    }

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 currentAllowance = _allowances[from][msg.sender];
        require(currentAllowance >= amount, "ERC20: insufficient allowance");
        _allowances[from][msg.sender] = currentAllowance - amount;
        _transfer(from, to, amount);
        return true;
    }

    function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "ERC20: transfer from zero address");
        require(to != address(0), "ERC20: transfer to zero address");
        require(_balances[from] >= amount, "ERC20: insufficient balance");

        _balances[from] -= amount;
        _balances[to] += amount;
        emit Transfer(from, to, amount);
    }

    function _mint(address account, uint256 amount) internal {
        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);
    }
}

// Использование интерфейса для взаимодействия
contract TokenInteraction {
    function getBalance(address token, address account) external view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }

    function safeTransfer(address token, address to, uint256 amount) external {
        bool success = IERC20(token).transfer(to, amount);
        require(success, "Transfer failed");
    }
}
```

## Абстрактные контракты

```solidity
// Абстрактный контракт - не может быть развёрнут напрямую
abstract contract PaymentProcessor {
    address public treasury;

    constructor(address _treasury) {
        treasury = _treasury;
    }

    // Абстрактная функция (без реализации)
    function processPayment(address from, uint256 amount) public virtual returns (bool);

    // Обычная функция с реализацией
    function getTreasury() public view returns (address) {
        return treasury;
    }

    // Виртуальная функция с реализацией по умолчанию
    function calculateFee(uint256 amount) public virtual pure returns (uint256) {
        return amount / 100; // 1% по умолчанию
    }
}

// Реализация абстрактного контракта
contract ETHPaymentProcessor is PaymentProcessor {
    constructor(address _treasury) PaymentProcessor(_treasury) {}

    function processPayment(address from, uint256 amount) public override returns (bool) {
        uint256 fee = calculateFee(amount);
        uint256 netAmount = amount - fee;

        // Логика обработки платежа
        (bool success, ) = treasury.call{value: fee}("");
        require(success, "Fee transfer failed");

        return true;
    }

    // Переопределение расчёта комиссии
    function calculateFee(uint256 amount) public pure override returns (uint256) {
        return amount * 2 / 100; // 2% для ETH
    }
}

// Частичная реализация (всё ещё абстрактный)
abstract contract TokenPaymentProcessor is PaymentProcessor {
    address public token;

    constructor(address _treasury, address _token) PaymentProcessor(_treasury) {
        token = _token;
    }

    function calculateFee(uint256 amount) public pure override returns (uint256) {
        return amount * 5 / 1000; // 0.5% для токенов
    }

    // processPayment всё ещё не реализован
}
```

## Библиотеки

```solidity
// Библиотека для работы с массивами
library ArrayUtils {
    // Функция для поиска элемента
    function indexOf(uint256[] storage arr, uint256 value) internal view returns (int256) {
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] == value) {
                return int256(i);
            }
        }
        return -1;
    }

    // Функция для удаления элемента
    function remove(uint256[] storage arr, uint256 index) internal {
        require(index < arr.length, "Index out of bounds");
        arr[index] = arr[arr.length - 1];
        arr.pop();
    }

    // Pure функция (не требует storage)
    function sum(uint256[] memory arr) internal pure returns (uint256 total) {
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i];
        }
    }
}

// Библиотека для безопасных математических операций
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        return a - b;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }
}

// Использование библиотек
contract LibraryExample {
    using ArrayUtils for uint256[];  // Присоединение к типу
    using SafeMath for uint256;

    uint256[] public numbers;

    function addNumber(uint256 num) public {
        numbers.push(num);
    }

    function findNumber(uint256 num) public view returns (int256) {
        return numbers.indexOf(num);  // Вызов как метод
    }

    function removeAt(uint256 index) public {
        numbers.remove(index);
    }

    function getSum() public view returns (uint256) {
        return ArrayUtils.sum(numbers);  // Прямой вызов
    }

    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        return a.add(b);  // SafeMath как метод
    }
}
```

## События (Events)

```solidity
contract EventsExample {
    // Базовое событие
    event Transfer(address from, address to, uint256 amount);

    // Индексированные параметры (до 3-х)
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    // Анонимное событие (без сигнатуры в topic[0])
    event AnonymousEvent(uint256 data) anonymous;

    // Сложное событие
    event OrderCreated(
        uint256 indexed orderId,
        address indexed buyer,
        address indexed seller,
        uint256 price,
        uint256 quantity,
        bytes32 productHash
    );

    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        balances[msg.sender] -= amount;
        balances[to] += amount;

        // Emit события
        emit Transfer(msg.sender, to, amount);
    }

    function approve(address spender, uint256 value) public {
        emit Approval(msg.sender, spender, value);
    }

    function createOrder(
        uint256 orderId,
        address seller,
        uint256 price,
        uint256 quantity,
        string memory product
    ) public {
        bytes32 productHash = keccak256(bytes(product));

        emit OrderCreated(
            orderId,
            msg.sender,
            seller,
            price,
            quantity,
            productHash
        );
    }
}

/*
Структура лога:
┌─────────────────────────────────────────────────────────────────┐
│                        Event Log                                │
├─────────────────────────────────────────────────────────────────┤
│ topics[0]: keccak256("Transfer(address,address,uint256)")      │
│ topics[1]: indexed from (если indexed)                         │
│ topics[2]: indexed to (если indexed)                           │
│ data: неиндексированные параметры (ABI encoded)                │
└─────────────────────────────────────────────────────────────────┘

Gas costs:
- Базовая стоимость: 375 gas
- За topic: 375 gas (до 4 topics)
- За байт data: 8 gas
*/
```

## Обработка ошибок

### require, revert, assert

```solidity
contract ErrorHandling {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // require - для проверки входных данных и условий
    function withdraw(uint256 amount) public {
        require(amount > 0, "Amount must be positive");
        require(balances[msg.sender] >= amount, "Insufficient balance");

        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    // revert - для сложной логики с условиями
    function transfer(address to, uint256 amount) public {
        if (to == address(0)) {
            revert("Cannot transfer to zero address");
        }
        if (amount == 0) {
            revert("Amount cannot be zero");
        }
        if (balances[msg.sender] < amount) {
            revert("Insufficient balance");
        }

        balances[msg.sender] -= amount;
        balances[to] += amount;
    }

    // assert - для проверки инвариантов (не должно никогда быть false)
    function internalCheck() internal view {
        // Используется для выявления багов в логике
        assert(balances[owner] >= 0); // Всегда должно быть true
    }
}
```

### Custom Errors (Solidity 0.8.4+)

```solidity
// Определение пользовательских ошибок
error Unauthorized(address caller, address required);
error InsufficientBalance(uint256 available, uint256 required);
error InvalidAddress(address addr);
error TransferFailed(address to, uint256 amount);
error ZeroAmount();

contract CustomErrorsExample {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender, owner);
        }
        _;
    }

    function withdraw(uint256 amount) public {
        // Проверка нулевой суммы
        if (amount == 0) {
            revert ZeroAmount();
        }

        // Проверка баланса
        uint256 balance = balances[msg.sender];
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }

        balances[msg.sender] -= amount;

        // Проверка перевода
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        if (!success) {
            revert TransferFailed(msg.sender, amount);
        }
    }

    function setOwner(address newOwner) public onlyOwner {
        if (newOwner == address(0)) {
            revert InvalidAddress(newOwner);
        }
        owner = newOwner;
    }
}

// Преимущества custom errors:
// 1. Экономия gas (~24 bytes vs полная строка)
// 2. Структурированные данные об ошибке
// 3. Лучше читаемость и поддержка
```

### try/catch

```solidity
interface IExternalContract {
    function riskyOperation() external returns (uint256);
    function getValue() external view returns (uint256);
}

contract TryCatchExample {
    event OperationSucceeded(uint256 result);
    event OperationFailed(string reason);
    event LowLevelError(bytes data);
    event PanicError(uint256 code);

    // Обработка внешнего вызова
    function safeCall(address target) public returns (bool success, uint256 result) {
        try IExternalContract(target).riskyOperation() returns (uint256 value) {
            // Успешное выполнение
            emit OperationSucceeded(value);
            return (true, value);
        } catch Error(string memory reason) {
            // revert с сообщением
            emit OperationFailed(reason);
            return (false, 0);
        } catch Panic(uint256 code) {
            // assert провал или arithmetic error
            emit PanicError(code);
            return (false, 0);
        } catch (bytes memory data) {
            // Другие ошибки (custom errors, out of gas и т.д.)
            emit LowLevelError(data);
            return (false, 0);
        }
    }

    // try/catch для создания контракта
    function safeCreateContract() public returns (address) {
        try new ChildContract(100) returns (ChildContract child) {
            return address(child);
        } catch {
            return address(0);
        }
    }

    // Декодирование custom error
    function decodeCustomError(bytes memory errorData) public pure returns (string memory) {
        // Селектор ошибки (первые 4 байта)
        bytes4 selector;
        assembly {
            selector := mload(add(errorData, 32))
        }

        // Проверка известных ошибок
        if (selector == InsufficientBalance.selector) {
            return "InsufficientBalance error";
        }
        return "Unknown error";
    }
}

contract ChildContract {
    uint256 public value;

    constructor(uint256 _value) {
        require(_value > 0, "Value must be positive");
        value = _value;
    }
}

// Коды Panic:
// 0x01: assert(false)
// 0x11: Arithmetic overflow/underflow
// 0x12: Division by zero
// 0x21: Invalid enum value
// 0x22: Storage encoding error
// 0x31: pop() on empty array
// 0x32: Array index out of bounds
// 0x41: Memory allocation error
// 0x51: Zero function pointer call
```

## Inline Assembly (Yul)

```solidity
contract AssemblyExamples {
    // Базовые операции
    function add(uint256 a, uint256 b) public pure returns (uint256 result) {
        assembly {
            result := add(a, b)
        }
    }

    // Прямой доступ к storage
    function getStorageAt(uint256 slot) public view returns (bytes32 value) {
        assembly {
            value := sload(slot)
        }
    }

    function setStorageAt(uint256 slot, bytes32 value) public {
        assembly {
            sstore(slot, value)
        }
    }

    // Работа с memory
    function memoryOperations() public pure returns (bytes32 result) {
        assembly {
            // Выделение памяти
            let ptr := mload(0x40)  // Free memory pointer

            // Запись в память
            mstore(ptr, 0x1234)

            // Обновление free memory pointer
            mstore(0x40, add(ptr, 32))

            // Чтение из памяти
            result := mload(ptr)
        }
    }

    // Проверка размера кода (is contract?)
    function isContract(address account) public view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }

    // Эффективное копирование
    function efficientCopy(bytes memory data) public pure returns (bytes memory) {
        bytes memory result = new bytes(data.length);

        assembly {
            let len := mload(data)
            let src := add(data, 32)
            let dst := add(result, 32)

            // Копирование по 32 байта
            for { let i := 0 } lt(i, len) { i := add(i, 32) } {
                mstore(add(dst, i), mload(add(src, i)))
            }
        }

        return result;
    }

    // Delegatecall через assembly
    function delegateCall(address target, bytes memory data)
        public
        returns (bool success, bytes memory returnData)
    {
        assembly {
            // Выполнение delegatecall
            success := delegatecall(
                gas(),
                target,
                add(data, 32),
                mload(data),
                0,
                0
            )

            // Получение размера возврата
            let size := returndatasize()

            // Выделение памяти для результата
            returnData := mload(0x40)
            mstore(0x40, add(returnData, add(size, 32)))
            mstore(returnData, size)

            // Копирование данных
            returndatacopy(add(returnData, 32), 0, size)
        }
    }

    // Кастомный selector
    function getSelector(string memory signature) public pure returns (bytes4) {
        bytes4 selector;
        assembly {
            // Вычисление keccak256 от сигнатуры
            selector := keccak256(add(signature, 32), mload(signature))
        }
        return selector;
    }

    // Оптимизация газа для массивов
    function sumArray(uint256[] memory arr) public pure returns (uint256 total) {
        assembly {
            let len := mload(arr)
            let dataPtr := add(arr, 32)

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                total := add(total, mload(add(dataPtr, mul(i, 32))))
            }
        }
    }
}

/*
Часто используемые opcodes:
┌─────────────┬──────────────────────────────────────────┐
│   Opcode    │              Описание                    │
├─────────────┼──────────────────────────────────────────┤
│ add(a, b)   │ Сложение                                 │
│ sub(a, b)   │ Вычитание                                │
│ mul(a, b)   │ Умножение                                │
│ div(a, b)   │ Деление                                  │
│ mod(a, b)   │ Остаток от деления                       │
│ mload(p)    │ Загрузка из memory                       │
│ mstore(p,v) │ Запись в memory                          │
│ sload(p)    │ Загрузка из storage                      │
│ sstore(p,v) │ Запись в storage                         │
│ calldataload│ Загрузка из calldata                     │
│ caller      │ msg.sender                               │
│ callvalue   │ msg.value                                │
│ gas         │ Оставшийся gas                           │
│ returndatasz│ Размер возвращённых данных               │
│ revert(p,s) │ Откат транзакции                         │
└─────────────┴──────────────────────────────────────────┘
*/
```

## Receive и Fallback функции

```solidity
contract ReceiveFallback {
    event Received(address sender, uint256 amount);
    event FallbackCalled(address sender, bytes data);

    // Вызывается при получении ETH без данных
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }

    // Вызывается при:
    // 1. Вызове несуществующей функции
    // 2. Отправке ETH с данными (если нет receive)
    // 3. Отправке ETH без данных (если нет receive)
    fallback() external payable {
        emit FallbackCalled(msg.sender, msg.data);
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

/*
Схема выбора функции:
┌─────────────────────────────────────────────────────────────────┐
│                 Входящий вызов                                  │
├─────────────────────────────────────────────────────────────────┤
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                           │
│              │  msg.data пустой?   │                           │
│              └──────────┬──────────┘                           │
│                   │           │                                 │
│                  YES         NO                                 │
│                   │           │                                 │
│                   ▼           ▼                                 │
│     ┌─────────────────┐  ┌─────────────────┐                   │
│     │ receive() есть? │  │ Функция найдена?│                   │
│     └────────┬────────┘  └────────┬────────┘                   │
│         │        │           │        │                         │
│        YES      NO          YES      NO                         │
│         │        │           │        │                         │
│         ▼        ▼           ▼        ▼                         │
│    receive()  fallback()  function  fallback()                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
*/
```

## Низкоуровневые вызовы

```solidity
contract LowLevelCalls {
    event CallResult(bool success, bytes data);

    // call - обычный внешний вызов
    function callContract(
        address target,
        bytes memory data
    ) public payable returns (bool, bytes memory) {
        (bool success, bytes memory result) = target.call{value: msg.value}(data);
        emit CallResult(success, result);
        return (success, result);
    }

    // staticcall - только для view/pure функций
    function staticCallContract(
        address target,
        bytes memory data
    ) public view returns (bool, bytes memory) {
        (bool success, bytes memory result) = target.staticcall(data);
        return (success, result);
    }

    // delegatecall - выполнение в контексте текущего контракта
    function delegateCallContract(
        address target,
        bytes memory data
    ) public returns (bool, bytes memory) {
        (bool success, bytes memory result) = target.delegatecall(data);
        return (success, result);
    }

    // Формирование calldata
    function encodeCall(uint256 value) public pure returns (bytes memory) {
        // Способ 1: abi.encodeWithSignature
        bytes memory data1 = abi.encodeWithSignature("setValue(uint256)", value);

        // Способ 2: abi.encodeWithSelector
        bytes memory data2 = abi.encodeWithSelector(bytes4(keccak256("setValue(uint256)")), value);

        // Способ 3: abi.encodeCall (type-safe, Solidity 0.8.11+)
        // bytes memory data3 = abi.encodeCall(ITarget.setValue, (value));

        return data1;
    }

    // Декодирование результата
    function decodeResult(bytes memory data) public pure returns (uint256) {
        return abi.decode(data, (uint256));
    }
}
```

## Best Practices

### Рекомендации по продвинутым концепциям

```solidity
contract BestPractices {
    // 1. Используйте интерфейсы для внешних вызовов
    IERC20 public token;

    // 2. Предпочитайте composition над наследованием
    // Плохо: contract MyToken is ERC20, Pausable, Ownable, AccessControl...
    // Хорошо: Использовать библиотеки и композицию

    // 3. Проверяйте результаты низкоуровневых вызовов
    function safeTransferETH(address to, uint256 amount) internal {
        (bool success, ) = to.call{value: amount}("");
        require(success, "ETH transfer failed");
    }

    // 4. Используйте custom errors вместо строк
    error TransferFailed(address to, uint256 amount);

    // 5. Документируйте виртуальные и override функции
    /// @notice Переопределите для кастомной логики
    /// @dev Вызывается при каждом трансфере
    function _beforeTransfer(address from, address to) internal virtual {}

    // 6. Будьте осторожны с delegatecall
    // - Storage layout должен совпадать
    // - Не используйте с непроверенными контрактами

    // 7. Ограничивайте использование assembly
    // - Только для критических оптимизаций
    // - Всегда комментируйте код
    // - Тщательно тестируйте
}
```

### Типичные ошибки

| Ошибка | Проблема | Решение |
|--------|----------|---------|
| Неправильный порядок наследования | Неожиданное поведение super | Следовать C3 linearization |
| Storage collision в delegatecall | Перезапись данных | Использовать одинаковый layout |
| Unchecked low-level call | Пропуск ошибок | Всегда проверять success |
| Избыточное наследование | Сложность, gas | Composition over inheritance |
| assert вместо require | Трата газа при ошибках | require для валидации |

## Заключение

Продвинутые концепции Solidity позволяют создавать сложные, модульные и эффективные смарт-контракты. Понимание наследования, интерфейсов, событий, обработки ошибок и низкоуровневого программирования критически важно для профессиональной разработки. В следующем разделе мы рассмотрим паттерны проектирования, которые помогают структурировать код и решать типичные задачи.

---

**Навигация:**
- [Обзор Solidity](../readme.md)
- [Основы Solidity](../01-basics/readme.md)
- [Паттерны проектирования](../03-patterns/readme.md)
