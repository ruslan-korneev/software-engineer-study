# Контроль доступа (Access Control)

Уязвимости контроля доступа возникают, когда критические функции контракта не имеют надлежащей защиты от несанкционированного использования. Неправильная реализация авторизации позволяет любому пользователю выполнять привилегированные операции: менять владельца контракта, выводить средства, изменять параметры протокола или даже уничтожать контракт.

## Типы уязвимостей контроля доступа

### 1. Отсутствие проверки авторизации

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ КОНТРАКТ - НЕ ИСПОЛЬЗОВАТЬ!
contract VulnerableWallet {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function deposit() external payable {}

    // УЯЗВИМОСТЬ: Нет проверки, что вызывающий — владелец
    function withdraw(uint256 amount) external {
        payable(msg.sender).transfer(amount);
    }

    // УЯЗВИМОСТЬ: Любой может стать владельцем
    function setOwner(address newOwner) external {
        owner = newOwner;
    }
}
```

### 2. Неправильное использование tx.origin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ КОНТРАКТ
contract VulnerableTxOrigin {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // УЯЗВИМОСТЬ: tx.origin вместо msg.sender
    function withdraw() external {
        // tx.origin — это оригинальный отправитель транзакции (EOA)
        // Атакующий контракт может обманом заставить владельца вызвать его функцию
        require(tx.origin == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
}

// Контракт атакующего
contract TxOriginAttacker {
    VulnerableTxOrigin public target;
    address public attacker;

    constructor(address _target) {
        target = VulnerableTxOrigin(_target);
        attacker = msg.sender;
    }

    // Владелец target случайно вызывает эту функцию
    function innocentLookingFunction() external {
        // tx.origin = владелец target (тот, кто инициировал транзакцию)
        // Средства переводятся на адрес владельца, но можно сделать хитрее
        target.withdraw();
    }
}
```

### 3. Неинициализированные прокси

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ Implementation контракт
contract VulnerableImplementation {
    address public owner;
    bool private initialized;

    // УЯЗВИМОСТЬ: initialize может быть вызван кем угодно
    function initialize(address _owner) external {
        require(!initialized, "Already initialized");
        owner = _owner;
        initialized = true;
    }

    function withdraw() external {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
}
```

### 4. Публичные внутренние функции

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// УЯЗВИМЫЙ КОНТРАКТ
contract VulnerableVisibility {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    // УЯЗВИМОСТЬ: Должна быть internal или private
    function _mint(address to, uint256 amount) public {
        balances[to] += amount;
    }

    // УЯЗВИМОСТЬ: Забыли modifier
    function adminMint(address to, uint256 amount) external {
        // Нет require(msg.sender == owner)!
        _mint(to, amount);
    }
}
```

## Реальные атаки

### Parity Wallet Hack (2017)

В июле 2017 года взлом Parity Multisig Wallet привёл к потере ~$30 млн ETH.

```solidity
// Упрощённая версия уязвимого кода Parity

// Библиотечный контракт
contract WalletLibrary {
    address public owner;

    // УЯЗВИМОСТЬ: initWallet была public и не защищена
    function initWallet(address _owner) public {
        owner = _owner;
    }

    function execute(address to, uint256 value, bytes memory data) public {
        require(msg.sender == owner, "Not owner");
        (bool success, ) = to.call{value: value}(data);
        require(success, "Call failed");
    }
}

// Кошелёк-прокси
contract Wallet {
    // Делегирует все вызовы в WalletLibrary
}
```

**Атака**:
1. Атакующий вызвал `initWallet()` напрямую на библиотечном контракте
2. Стал владельцем библиотеки
3. Вызвал `execute()` для вывода средств

### Parity Wallet Freeze (ноябрь 2017)

Второй инцидент с Parity заморозил ~$150 млн ETH навсегда:

```solidity
// Атакующий вызвал initWallet() на библиотеке
// Затем вызвал selfdestruct()
// Все кошельки, использующие эту библиотеку, перестали работать
```

### Poly Network Hack (2021)

Крупнейший DeFi взлом ($610 млн) был связан с уязвимостью в управлении ролями при кросс-чейн взаимодействии.

## Безопасные паттерны

### 1. Модификаторы доступа

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SecureOwnership {
    address public owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not the owner");
        _;
    }

    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "New owner is zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function withdraw() external onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    function deposit() external payable {}
}
```

### 2. OpenZeppelin Ownable

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
    constructor() Ownable(msg.sender) {}

    function adminFunction() external onlyOwner {
        // Только владелец может вызвать
    }

    // Встроенные функции:
    // - transferOwnership(address newOwner)
    // - renounceOwnership()
    // - owner() view returns (address)
}
```

### 3. OpenZeppelin Ownable2Step

Двухэтапная передача владения для предотвращения ошибок:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable2Step.sol";

contract SecureContract is Ownable2Step {
    constructor() Ownable(msg.sender) {}

    // Передача владения:
    // 1. Текущий владелец вызывает transferOwnership(newOwner)
    // 2. Новый владелец вызывает acceptOwnership()
    // Если newOwner — неправильный адрес, владение не потеряется
}
```

### 4. Role-Based Access Control (RBAC)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract Treasury is AccessControl {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant TREASURER_ROLE = keccak256("TREASURER_ROLE");
    bytes32 public constant AUDITOR_ROLE = keccak256("AUDITOR_ROLE");

    uint256 public constant MAX_WITHDRAWAL = 100 ether;

    mapping(bytes32 => uint256) public withdrawalLimits;

    event Withdrawal(address indexed by, uint256 amount);

    constructor() {
        // DEFAULT_ADMIN_ROLE может управлять всеми ролями
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ADMIN_ROLE, msg.sender);

        // Устанавливаем лимиты для ролей
        withdrawalLimits[ADMIN_ROLE] = type(uint256).max;
        withdrawalLimits[TREASURER_ROLE] = 10 ether;
    }

    function deposit() external payable {}

    function withdraw(uint256 amount) external {
        require(
            hasRole(ADMIN_ROLE, msg.sender) || hasRole(TREASURER_ROLE, msg.sender),
            "Not authorized"
        );

        bytes32 role = hasRole(ADMIN_ROLE, msg.sender) ? ADMIN_ROLE : TREASURER_ROLE;
        require(amount <= withdrawalLimits[role], "Exceeds limit");

        payable(msg.sender).transfer(amount);
        emit Withdrawal(msg.sender, amount);
    }

    function setWithdrawalLimit(bytes32 role, uint256 limit) external onlyRole(ADMIN_ROLE) {
        withdrawalLimits[role] = limit;
    }

    // Только аудитор может просматривать историю
    function getAuditLog() external view onlyRole(AUDITOR_ROLE) returns (string memory) {
        return "Audit data";
    }
}
```

### 5. Иерархия ролей с AccessControlEnumerable

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/extensions/AccessControlEnumerable.sol";

contract EnumerableRoles is AccessControlEnumerable {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function getMinterCount() external view returns (uint256) {
        return getRoleMemberCount(MINTER_ROLE);
    }

    function getMinterAt(uint256 index) external view returns (address) {
        return getRoleMember(MINTER_ROLE, index);
    }

    function getAllMinters() external view returns (address[] memory) {
        uint256 count = getRoleMemberCount(MINTER_ROLE);
        address[] memory minters = new address[](count);

        for (uint256 i = 0; i < count; i++) {
            minters[i] = getRoleMember(MINTER_ROLE, i);
        }

        return minters;
    }
}
```

### 6. Timelock для критических операций

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";

contract TimelockAdmin is Ownable {
    uint256 public constant TIMELOCK_DURATION = 2 days;

    struct PendingChange {
        bytes32 operationHash;
        uint256 executeAfter;
        bool executed;
    }

    mapping(bytes32 => PendingChange) public pendingChanges;

    event ChangeScheduled(bytes32 indexed operationId, uint256 executeAfter);
    event ChangeExecuted(bytes32 indexed operationId);
    event ChangeCancelled(bytes32 indexed operationId);

    constructor() Ownable(msg.sender) {}

    function scheduleWithdrawal(address to, uint256 amount) external onlyOwner returns (bytes32) {
        bytes32 operationId = keccak256(abi.encodePacked(to, amount, block.timestamp));

        pendingChanges[operationId] = PendingChange({
            operationHash: keccak256(abi.encodePacked(to, amount)),
            executeAfter: block.timestamp + TIMELOCK_DURATION,
            executed: false
        });

        emit ChangeScheduled(operationId, pendingChanges[operationId].executeAfter);
        return operationId;
    }

    function executeWithdrawal(bytes32 operationId, address to, uint256 amount) external onlyOwner {
        PendingChange storage change = pendingChanges[operationId];

        require(change.executeAfter != 0, "Operation not scheduled");
        require(block.timestamp >= change.executeAfter, "Timelock not expired");
        require(!change.executed, "Already executed");
        require(
            change.operationHash == keccak256(abi.encodePacked(to, amount)),
            "Parameters mismatch"
        );

        change.executed = true;
        payable(to).transfer(amount);

        emit ChangeExecuted(operationId);
    }

    function cancelChange(bytes32 operationId) external onlyOwner {
        require(pendingChanges[operationId].executeAfter != 0, "Not scheduled");
        require(!pendingChanges[operationId].executed, "Already executed");

        delete pendingChanges[operationId];
        emit ChangeCancelled(operationId);
    }

    receive() external payable {}
}
```

### 7. Multisig защита

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleMultisig {
    address[] public owners;
    uint256 public required;

    struct Transaction {
        address to;
        uint256 value;
        bytes data;
        bool executed;
        uint256 confirmations;
    }

    Transaction[] public transactions;
    mapping(uint256 => mapping(address => bool)) public confirmations;
    mapping(address => bool) public isOwner;

    modifier onlyOwner() {
        require(isOwner[msg.sender], "Not an owner");
        _;
    }

    modifier txExists(uint256 txId) {
        require(txId < transactions.length, "Tx does not exist");
        _;
    }

    modifier notExecuted(uint256 txId) {
        require(!transactions[txId].executed, "Already executed");
        _;
    }

    modifier notConfirmed(uint256 txId) {
        require(!confirmations[txId][msg.sender], "Already confirmed");
        _;
    }

    constructor(address[] memory _owners, uint256 _required) {
        require(_owners.length > 0, "Owners required");
        require(_required > 0 && _required <= _owners.length, "Invalid required");

        for (uint256 i = 0; i < _owners.length; i++) {
            address owner = _owners[i];
            require(owner != address(0), "Invalid owner");
            require(!isOwner[owner], "Duplicate owner");

            isOwner[owner] = true;
            owners.push(owner);
        }

        required = _required;
    }

    function submitTransaction(address _to, uint256 _value, bytes calldata _data)
        external
        onlyOwner
        returns (uint256)
    {
        uint256 txId = transactions.length;

        transactions.push(Transaction({
            to: _to,
            value: _value,
            data: _data,
            executed: false,
            confirmations: 0
        }));

        return txId;
    }

    function confirmTransaction(uint256 _txId)
        external
        onlyOwner
        txExists(_txId)
        notExecuted(_txId)
        notConfirmed(_txId)
    {
        Transaction storage transaction = transactions[_txId];
        confirmations[_txId][msg.sender] = true;
        transaction.confirmations++;
    }

    function executeTransaction(uint256 _txId)
        external
        onlyOwner
        txExists(_txId)
        notExecuted(_txId)
    {
        Transaction storage transaction = transactions[_txId];
        require(transaction.confirmations >= required, "Not enough confirmations");

        transaction.executed = true;

        (bool success, ) = transaction.to.call{value: transaction.value}(transaction.data);
        require(success, "Tx failed");
    }

    receive() external payable {}
}
```

## Безопасная инициализация прокси

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract SecureUpgradeable is Initializable, OwnableUpgradeable {
    uint256 public value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers(); // Защита implementation от инициализации
    }

    function initialize(address initialOwner) public initializer {
        __Ownable_init(initialOwner);
        value = 100;
    }

    function setValue(uint256 _value) external onlyOwner {
        value = _value;
    }
}
```

## Правильное использование visibility

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VisibilityBestPractices {
    address public owner; // Чтение доступно всем
    uint256 private secretValue; // Только внутри контракта
    mapping(address => uint256) internal balances; // Доступно наследникам

    // external — только внешние вызовы (эффективнее для calldata)
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // public — внешние и внутренние вызовы
    function getBalance(address user) public view returns (uint256) {
        return balances[user];
    }

    // internal — только внутри контракта и наследников
    function _updateBalance(address user, uint256 amount) internal {
        balances[user] = amount;
    }

    // private — только внутри этого контракта
    function _validateOwner() private view {
        require(msg.sender == owner, "Not owner");
    }
}
```

## Best Practices

### 1. Всегда используйте модификаторы

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

modifier onlyRole(bytes32 role) {
    require(hasRole(role, msg.sender), "Missing role");
    _;
}
```

### 2. Никогда не используйте tx.origin для авторизации

```solidity
// ПЛОХО
require(tx.origin == owner);

// ХОРОШО
require(msg.sender == owner);
```

### 3. Используйте двухэтапную передачу владения

```solidity
// Предотвращает случайную передачу на неправильный адрес
function transferOwnership(address newOwner) external onlyOwner {
    pendingOwner = newOwner;
}

function acceptOwnership() external {
    require(msg.sender == pendingOwner);
    owner = pendingOwner;
    pendingOwner = address(0);
}
```

### 4. Timelock для критических операций

```solidity
// Дайте пользователям время отреагировать на изменения
uint256 public constant TIMELOCK = 48 hours;
```

### 5. События для всех изменений прав

```solidity
event RoleGranted(bytes32 role, address account, address sender);
event RoleRevoked(bytes32 role, address account, address sender);
event OwnershipTransferred(address previousOwner, address newOwner);
```

### 6. Проверяйте нулевые адреса

```solidity
function setAdmin(address newAdmin) external onlyOwner {
    require(newAdmin != address(0), "Zero address");
    admin = newAdmin;
}
```

### 7. Минимальные привилегии

```solidity
// Вместо одной роли "admin" со всеми правами
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
```

## Инструменты для аудита

### Slither

```bash
slither . --detect unprotected-upgrade,arbitrary-send-erc20,suicidal
```

### Проверка visibility

```bash
slither . --print function-summary
```

## Checklist безопасности

- [ ] Все критические функции имеют модификаторы доступа
- [ ] tx.origin не используется для авторизации
- [ ] Initializers защищены модификатором initializer
- [ ] Implementation контракты имеют _disableInitializers()
- [ ] Внутренние функции имеют правильную visibility (internal/private)
- [ ] Двухэтапная передача владения для критических ролей
- [ ] Timelock для опасных операций
- [ ] События для всех изменений прав доступа
- [ ] Проверка на нулевые адреса при назначении ролей
- [ ] Multisig для высокорисковых операций

## Заключение

Контроль доступа — фундаментальный аспект безопасности смарт-контрактов. Неправильная авторизация может привести к полной потере средств, как показали атаки на Parity Wallet и другие протоколы. Используйте проверенные библиотеки (OpenZeppelin), следуйте принципу минимальных привилегий и всегда проводите тщательный аудит функций с ограниченным доступом.

Помните: в мире смарт-контрактов одна забытая проверка `onlyOwner` может стоить миллионы долларов.
