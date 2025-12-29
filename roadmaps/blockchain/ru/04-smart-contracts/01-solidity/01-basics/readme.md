# Основы Solidity

## Введение

Основы Solidity включают понимание типов данных, переменных, функций и управляющих конструкций. Эти базовые концепции формируют фундамент для написания смарт-контрактов.

## Структура контракта

```solidity
// SPDX-License-Identifier: MIT      // 1. Лицензия (обязательно)
pragma solidity ^0.8.20;              // 2. Версия компилятора

import "./OtherContract.sol";         // 3. Импорты

// 4. Определение контракта
contract MyContract {
    // 5. Переменные состояния
    uint256 public value;

    // 6. События
    event ValueChanged(uint256 newValue);

    // 7. Модификаторы
    modifier onlyPositive(uint256 _value) {
        require(_value > 0, "Must be positive");
        _;
    }

    // 8. Конструктор
    constructor(uint256 _initialValue) {
        value = _initialValue;
    }

    // 9. Функции
    function setValue(uint256 _value) public onlyPositive(_value) {
        value = _value;
        emit ValueChanged(_value);
    }
}
```

## Типы данных

### Схема типов данных

```
┌─────────────────────────────────────────────────────────────────┐
│                      Типы данных Solidity                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Value Types (Значимые)          Reference Types (Ссылочные)   │
│  ├── bool                        ├── arrays                     │
│  ├── int/uint (8-256)            ├── struct                     │
│  ├── address                     ├── mapping                    │
│  ├── bytes1-bytes32              └── string                     │
│  ├── enum                                                       │
│  └── function                                                   │
│                                                                 │
│  Data Location (Расположение данных)                           │
│  ├── storage  - постоянное хранилище (на блокчейне)            │
│  ├── memory   - временная память (в рамках вызова)             │
│  └── calldata - только для чтения (входные параметры)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Целочисленные типы

```solidity
contract IntegerTypes {
    // Беззнаковые целые (uint)
    uint8 public tiny = 255;              // 0 до 2^8 - 1
    uint16 public small = 65535;          // 0 до 2^16 - 1
    uint256 public big = type(uint256).max; // 0 до 2^256 - 1
    uint public defaultUint;              // Алиас для uint256

    // Знаковые целые (int)
    int8 public signedTiny = -128;        // -2^7 до 2^7 - 1
    int256 public signedBig;              // -2^255 до 2^255 - 1
    int public defaultInt;                // Алиас для int256

    // Арифметические операции
    function arithmetic() public pure returns (uint256) {
        uint256 a = 10;
        uint256 b = 3;

        uint256 sum = a + b;       // 13
        uint256 diff = a - b;      // 7
        uint256 product = a * b;   // 30
        uint256 quotient = a / b;  // 3 (целочисленное деление)
        uint256 remainder = a % b; // 1 (остаток)
        uint256 power = a ** 2;    // 100

        return sum;
    }

    // Сравнение
    function compare(uint256 a, uint256 b) public pure returns (bool) {
        return a == b;  // равно
        // a != b  - не равно
        // a < b   - меньше
        // a <= b  - меньше или равно
        // a > b   - больше
        // a >= b  - больше или равно
    }
}
```

### Boolean (Логический тип)

```solidity
contract BooleanType {
    bool public isActive = true;
    bool public isComplete = false;

    function logicalOperations() public pure returns (bool) {
        bool a = true;
        bool b = false;

        bool andResult = a && b;  // false (И)
        bool orResult = a || b;   // true (ИЛИ)
        bool notResult = !a;      // false (НЕ)

        return andResult;
    }

    // Тернарный оператор
    function ternary(uint256 value) public pure returns (string memory) {
        return value > 100 ? "Big" : "Small";
    }
}
```

### Address (Адрес)

```solidity
contract AddressType {
    // Обычный адрес
    address public wallet = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;

    // Адрес, который может получать ETH
    address payable public payableWallet;

    // Специальные адреса
    address public zeroAddress = address(0);
    address public thisContract = address(this);

    constructor() {
        payableWallet = payable(msg.sender);
    }

    // Свойства адреса
    function getAddressInfo(address addr) public view returns (uint256 balance, uint256 codeSize) {
        balance = addr.balance;  // Баланс в wei

        // Размер кода (0 для EOA, >0 для контрактов)
        assembly {
            codeSize := extcodesize(addr)
        }
    }

    // Отправка ETH
    function sendETH(address payable to) public payable {
        // Способ 1: transfer (2300 gas, revert при ошибке)
        to.transfer(msg.value);

        // Способ 2: send (2300 gas, возвращает bool)
        bool success = to.send(msg.value);
        require(success, "Send failed");

        // Способ 3: call (рекомендуемый)
        (bool callSuccess, ) = to.call{value: msg.value}("");
        require(callSuccess, "Call failed");
    }
}
```

### Bytes и String

```solidity
contract BytesAndString {
    // Фиксированные байты (1-32 байта)
    bytes1 public oneByte = 0x42;        // 1 байт
    bytes32 public hash;                  // 32 байта (часто для хешей)

    // Динамические байты
    bytes public dynamicBytes = hex"001122";

    // Строки (UTF-8)
    string public greeting = "Привет!";

    // Конкатенация строк (Solidity 0.8.12+)
    function concat(string memory a, string memory b) public pure returns (string memory) {
        return string.concat(a, " ", b);
    }

    // Длина строки (в байтах, не символах!)
    function stringLength(string memory s) public pure returns (uint256) {
        return bytes(s).length;
    }

    // Сравнение строк
    function compareStrings(string memory a, string memory b) public pure returns (bool) {
        return keccak256(bytes(a)) == keccak256(bytes(b));
    }

    // Преобразование bytes32 в строку
    function bytes32ToString(bytes32 _bytes32) public pure returns (string memory) {
        uint8 i = 0;
        while(i < 32 && _bytes32[i] != 0) {
            i++;
        }
        bytes memory bytesArray = new bytes(i);
        for (uint8 j = 0; j < i; j++) {
            bytesArray[j] = _bytes32[j];
        }
        return string(bytesArray);
    }
}
```

### Arrays (Массивы)

```solidity
contract Arrays {
    // Фиксированный массив
    uint256[5] public fixedArray = [1, 2, 3, 4, 5];

    // Динамический массив
    uint256[] public dynamicArray;

    // Массив адресов
    address[] public participants;

    // Операции с массивами
    function arrayOperations() public {
        // Добавление элемента
        dynamicArray.push(100);

        // Удаление последнего элемента
        dynamicArray.pop();

        // Длина массива
        uint256 len = dynamicArray.length;

        // Доступ по индексу
        if (len > 0) {
            uint256 first = dynamicArray[0];
        }

        // Удаление по индексу (заменяет на 0, не уменьшает длину)
        delete dynamicArray[0];
    }

    // Создание массива в памяти
    function createMemoryArray(uint256 size) public pure returns (uint256[] memory) {
        uint256[] memory arr = new uint256[](size);
        for (uint256 i = 0; i < size; i++) {
            arr[i] = i * 2;
        }
        return arr;
    }

    // Эффективное удаление (swap and pop)
    function removeEfficient(uint256 index) public {
        require(index < dynamicArray.length, "Index out of bounds");

        // Меняем местами с последним элементом
        dynamicArray[index] = dynamicArray[dynamicArray.length - 1];
        // Удаляем последний
        dynamicArray.pop();
    }
}
```

### Struct (Структуры)

```solidity
contract Structs {
    // Определение структуры
    struct User {
        uint256 id;
        string name;
        address wallet;
        bool isActive;
        uint256 balance;
    }

    // Хранение структур
    User public admin;
    User[] public users;
    mapping(address => User) public userByAddress;

    // Создание структуры
    function createUser(string memory _name) public returns (uint256) {
        uint256 newId = users.length;

        // Способ 1: Позиционные аргументы
        User memory user1 = User(newId, _name, msg.sender, true, 0);

        // Способ 2: Именованные аргументы (рекомендуется)
        User memory user2 = User({
            id: newId,
            name: _name,
            wallet: msg.sender,
            isActive: true,
            balance: 0
        });

        // Способ 3: Прямое присвоение в storage
        users.push();
        User storage newUser = users[users.length - 1];
        newUser.id = newId;
        newUser.name = _name;
        newUser.wallet = msg.sender;
        newUser.isActive = true;

        userByAddress[msg.sender] = user2;

        return newId;
    }

    // Изменение структуры
    function updateUserBalance(address _wallet, uint256 _amount) public {
        // Storage reference для модификации
        User storage user = userByAddress[_wallet];
        user.balance += _amount;
    }
}
```

### Mapping (Отображения)

```solidity
contract Mappings {
    // Простой mapping
    mapping(address => uint256) public balances;

    // Mapping к структуре
    mapping(uint256 => User) public usersById;

    // Вложенные mappings
    mapping(address => mapping(address => uint256)) public allowances;

    // Mapping с массивом
    mapping(address => uint256[]) public userTransactions;

    struct User {
        string name;
        bool exists;
    }

    // Операции с mapping
    function mappingOperations() public {
        // Запись
        balances[msg.sender] = 100;

        // Чтение (несуществующий ключ вернёт значение по умолчанию)
        uint256 balance = balances[msg.sender];
        uint256 zeroBalance = balances[address(0)]; // 0

        // Удаление (сбрасывает к значению по умолчанию)
        delete balances[msg.sender];

        // Проверка существования (через дополнительное поле)
        usersById[1] = User("Alice", true);
        bool exists = usersById[1].exists;

        // Вложенный mapping
        allowances[msg.sender][address(this)] = 1000;
    }

    // Итерация по mapping (невозможна напрямую!)
    // Нужно хранить ключи отдельно
    address[] public allUsers;

    function addUser(address user) public {
        if (balances[user] == 0) {
            allUsers.push(user);
        }
        balances[user] = 100;
    }
}
```

### Enum (Перечисления)

```solidity
contract Enums {
    // Определение enum
    enum Status {
        Pending,    // 0
        Active,     // 1
        Completed,  // 2
        Cancelled   // 3
    }

    enum OrderType { Market, Limit, StopLoss }

    // Использование
    Status public currentStatus = Status.Pending;
    OrderType public orderType;

    // Изменение статуса
    function activate() public {
        currentStatus = Status.Active;
    }

    // Сравнение
    function isActive() public view returns (bool) {
        return currentStatus == Status.Active;
    }

    // Конвертация в uint
    function getStatusNumber() public view returns (uint8) {
        return uint8(currentStatus);
    }

    // Конвертация из uint
    function setStatus(uint8 _status) public {
        require(_status <= uint8(Status.Cancelled), "Invalid status");
        currentStatus = Status(_status);
    }

    // Min/Max значения
    function getMinMax() public pure returns (Status min, Status max) {
        min = type(Status).min; // Pending (0)
        max = type(Status).max; // Cancelled (3)
    }
}
```

## Переменные

### Области видимости переменных

```
┌─────────────────────────────────────────────────────────────────┐
│                    Области видимости                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  State Variables (Переменные состояния)                        │
│  └── Хранятся в storage (на блокчейне)                         │
│  └── Стоят gas для записи                                      │
│                                                                 │
│  Local Variables (Локальные переменные)                        │
│  └── Существуют только во время выполнения функции             │
│  └── Хранятся в memory или stack                               │
│                                                                 │
│  Global Variables (Глобальные переменные)                      │
│  └── Предоставляют информацию о блокчейне                      │
│  └── msg, block, tx и др.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### State Variables

```solidity
contract StateVariables {
    // Видимость переменных состояния
    uint256 public publicVar = 1;    // Автоматический getter
    uint256 internal internalVar = 2; // Доступно в контракте и наследниках
    uint256 private privateVar = 3;   // Только в этом контракте

    // Константы (должны быть известны во время компиляции)
    uint256 public constant MAX_SUPPLY = 1000000;
    address public constant BURN_ADDRESS = address(0);

    // Immutable (устанавливаются один раз в конструкторе)
    address public immutable owner;
    uint256 public immutable deployTime;

    constructor() {
        owner = msg.sender;
        deployTime = block.timestamp;
    }

    // Gas сравнение:
    // constant: 0 gas (значение подставляется в байткод)
    // immutable: 0 gas (значение подставляется в байткод после деплоя)
    // storage: 2100 gas (SLOAD) первое чтение, 100 gas последующие
}
```

### Global Variables

```solidity
contract GlobalVariables {
    // Информация о сообщении (msg)
    function messageInfo() public payable returns (
        address sender,
        uint256 value,
        bytes4 sig,
        bytes memory data
    ) {
        sender = msg.sender;    // Адрес отправителя
        value = msg.value;      // Отправленные wei
        sig = msg.sig;          // Первые 4 байта calldata (селектор функции)
        data = msg.data;        // Полные calldata
    }

    // Информация о блоке (block)
    function blockInfo() public view returns (
        uint256 number,
        uint256 timestamp,
        uint256 basefee,
        uint256 chainId,
        address coinbase,
        uint256 difficulty,
        uint256 gaslimit
    ) {
        number = block.number;       // Номер блока
        timestamp = block.timestamp; // Unix timestamp
        basefee = block.basefee;     // Базовая комиссия (EIP-1559)
        chainId = block.chainid;     // ID сети
        coinbase = block.coinbase;   // Адрес майнера/валидатора
        difficulty = block.prevrandao; // Случайное число (после Merge)
        gaslimit = block.gaslimit;   // Лимит газа блока
    }

    // Информация о транзакции (tx)
    function txInfo() public view returns (
        address origin,
        uint256 gasprice
    ) {
        origin = tx.origin;      // Исходный отправитель (EOA)
        gasprice = tx.gasprice;  // Цена газа транзакции
    }

    // Информация о gas
    function gasInfo() public view returns (uint256 remaining) {
        remaining = gasleft();   // Оставшийся gas
    }
}
```

## Функции

### Определение функций

```solidity
contract Functions {
    uint256 public value;

    // Базовая структура функции
    function functionName(uint256 param1, string memory param2)
        public           // видимость
        payable          // может принимать ETH
        returns (uint256, bool)  // возвращаемые значения
    {
        // тело функции
        return (param1, true);
    }
}
```

### Видимость функций

```solidity
contract FunctionVisibility {
    // public - доступна везде
    function publicFunction() public pure returns (string memory) {
        return "Anyone can call me";
    }

    // external - только снаружи контракта
    function externalFunction() external pure returns (string memory) {
        return "Only external calls";
    }

    // internal - контракт и наследники
    function internalFunction() internal pure returns (string memory) {
        return "Contract and children";
    }

    // private - только этот контракт
    function privateFunction() private pure returns (string memory) {
        return "Only this contract";
    }

    function callFunctions() public view {
        // Можно вызвать
        publicFunction();
        internalFunction();
        privateFunction();

        // Нельзя вызвать external изнутри напрямую
        // externalFunction(); // Ошибка!

        // Но можно через this (не рекомендуется)
        this.externalFunction();
    }
}

contract Child is FunctionVisibility {
    function childCall() public view {
        publicFunction();    // OK
        internalFunction();  // OK
        // privateFunction(); // Ошибка!
    }
}
```

### State Mutability

```solidity
contract StateMutability {
    uint256 public value = 100;

    // view - только чтение состояния
    function getValue() public view returns (uint256) {
        return value; // Можно читать state
    }

    // pure - не читает и не изменяет состояние
    function calculate(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b; // Только вычисления
    }

    // Без модификатора - может изменять состояние
    function setValue(uint256 _value) public {
        value = _value;
    }

    // payable - может принимать ETH
    function deposit() public payable {
        // msg.value содержит отправленные wei
    }
}
```

### Возвращаемые значения

```solidity
contract ReturnValues {
    // Один возвращаемый параметр
    function single() public pure returns (uint256) {
        return 42;
    }

    // Несколько возвращаемых параметров
    function multiple() public pure returns (uint256, bool, string memory) {
        return (42, true, "Hello");
    }

    // Именованные возвращаемые параметры
    function named() public pure returns (uint256 value, bool success) {
        value = 42;
        success = true;
        // return не обязателен
    }

    // Деструктуризация
    function useReturns() public pure {
        // Получить все значения
        (uint256 a, bool b, string memory c) = multiple();

        // Пропустить ненужные
        (uint256 value, , ) = multiple();

        // Именованные
        (uint256 v, bool s) = named();
    }
}
```

## Модификаторы доступа

```solidity
contract Modifiers {
    address public owner;
    bool public paused;
    mapping(address => bool) public admins;

    constructor() {
        owner = msg.sender;
        admins[msg.sender] = true;
    }

    // Базовый модификатор
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _; // Место вставки кода функции
    }

    // Модификатор с параметром
    modifier onlyRole(string memory role) {
        if (keccak256(bytes(role)) == keccak256("admin")) {
            require(admins[msg.sender], "Not admin");
        }
        _;
    }

    // Модификатор с проверкой после выполнения
    modifier noReentrancy() {
        require(!paused, "Paused");
        paused = true;
        _;
        paused = false;
    }

    // Множественные модификаторы (выполняются слева направо)
    function criticalAction() public onlyOwner noReentrancy {
        // Код функции
    }

    // Модификатор с возвратом
    modifier validAddress(address _addr) {
        require(_addr != address(0), "Zero address");
        require(_addr != address(this), "Cannot be this contract");
        _;
    }

    function transfer(address to) public validAddress(to) {
        // Перевод
    }
}
```

## Управляющие конструкции

### Условные операторы

```solidity
contract Conditionals {
    // if-else
    function checkValue(uint256 value) public pure returns (string memory) {
        if (value == 0) {
            return "Zero";
        } else if (value < 10) {
            return "Small";
        } else if (value < 100) {
            return "Medium";
        } else {
            return "Large";
        }
    }

    // Тернарный оператор
    function max(uint256 a, uint256 b) public pure returns (uint256) {
        return a > b ? a : b;
    }
}
```

### Циклы

```solidity
contract Loops {
    uint256[] public numbers;

    // for loop
    function forLoop(uint256 count) public pure returns (uint256 sum) {
        for (uint256 i = 0; i < count; i++) {
            sum += i;
        }
    }

    // while loop
    function whileLoop(uint256 count) public pure returns (uint256 sum) {
        uint256 i = 0;
        while (i < count) {
            sum += i;
            i++;
        }
    }

    // do-while loop
    function doWhileLoop(uint256 count) public pure returns (uint256 sum) {
        uint256 i = 0;
        do {
            sum += i;
            i++;
        } while (i < count);
    }

    // break и continue
    function breakContinue() public pure returns (uint256 sum) {
        for (uint256 i = 0; i < 100; i++) {
            if (i == 50) break;      // Выход из цикла
            if (i % 2 == 0) continue; // Пропуск итерации
            sum += i;
        }
    }

    // ВНИМАНИЕ: Избегайте unbounded loops!
    // function dangerousLoop() public {
    //     for (uint256 i = 0; i < numbers.length; i++) {
    //         // Может превысить gas limit!
    //     }
    // }

    // Безопасный паттерн с ограничением
    function safeLoop(uint256 maxIterations) public view returns (uint256 sum) {
        uint256 length = numbers.length;
        uint256 iterations = length < maxIterations ? length : maxIterations;

        for (uint256 i = 0; i < iterations; i++) {
            sum += numbers[i];
        }
    }
}
```

## Best Practices

### Рекомендации по типам данных

```solidity
contract BestPractices {
    // 1. Используйте uint256 по умолчанию (более эффективно на EVM)
    uint256 public amount; // Лучше чем uint8, uint16 и т.д.

    // 2. Упаковка storage переменных
    // Плохо (3 слота):
    uint128 a;  // Слот 0
    uint256 b;  // Слот 1 (не помещается в слот 0)
    uint128 c;  // Слот 2

    // Хорошо (2 слота):
    uint128 x;  // Слот 0 (первая половина)
    uint128 y;  // Слот 0 (вторая половина)
    uint256 z;  // Слот 1

    // 3. Используйте calldata для входных массивов
    function processArray(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
        return sum;
    }

    // 4. Кешируйте storage в memory для множественного доступа
    uint256[] public values;

    function sumValues() public view returns (uint256 sum) {
        uint256[] memory cached = values; // Одно чтение из storage
        for (uint256 i = 0; i < cached.length; i++) {
            sum += cached[i]; // Чтение из memory
        }
    }

    // 5. Используйте события вместо storage где возможно
    event DataStored(address indexed user, uint256 data, uint256 timestamp);

    function storeWithEvent(uint256 data) public {
        emit DataStored(msg.sender, data, block.timestamp);
        // Данные в event дешевле, чем в storage
    }
}
```

### Типичные ошибки

```solidity
contract CommonMistakes {
    mapping(address => uint256) public balances;

    // Ошибка 1: Использование tx.origin для авторизации
    // function badAuth() public {
    //     require(tx.origin == owner); // Уязвимо!
    // }

    // Правильно:
    address public owner;
    function goodAuth() public view {
        require(msg.sender == owner, "Not owner");
    }

    // Ошибка 2: Не проверять возврат send/transfer
    // function badSend(address payable to) public {
    //     to.send(1 ether); // Может молча провалиться!
    // }

    // Правильно:
    function goodSend(address payable to) public {
        (bool success, ) = to.call{value: 1 ether}("");
        require(success, "Transfer failed");
    }

    // Ошибка 3: Integer overflow (до Solidity 0.8.0)
    // uint8 x = 255;
    // x++; // Было бы 0 в 0.7.x

    // В 0.8.x автоматическая защита
    // Для намеренного переполнения используйте unchecked:
    function uncheckedOverflow() public pure returns (uint8) {
        unchecked {
            uint8 x = 255;
            return x + 1; // 0
        }
    }

    // Ошибка 4: Забыть about zero address
    function setOwner(address newOwner) public {
        require(newOwner != address(0), "Zero address");
        owner = newOwner;
    }
}
```

## Заключение

Основы Solidity — это фундамент для разработки смарт-контрактов. Понимание типов данных, их расположения в памяти, видимости переменных и функций критически важно для написания безопасного и эффективного кода. В следующих разделах мы рассмотрим продвинутые темы: наследование, интерфейсы и паттерны проектирования.

---

**Навигация:**
- [Обзор Solidity](../readme.md)
- [Продвинутый Solidity](../02-advanced/readme.md)
- [Паттерны проектирования](../03-patterns/readme.md)
