# Паттерны проектирования Solidity

## Введение

Паттерны проектирования в Solidity — это проверенные решения типичных задач при разработке смарт-контрактов. Они помогают создавать безопасный, поддерживаемый и gas-эффективный код.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Категории паттернов Solidity                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Создание                    Безопасность                       │
│  ├── Factory                 ├── Access Control                 │
│  ├── Clone (Minimal Proxy)   ├── Reentrancy Guard               │
│  └── Create2                 ├── Pausable                       │
│                              └── Pull Payment                   │
│                                                                 │
│  Обновляемость               Поведение                          │
│  ├── Proxy (UUPS, Transparent)  ├── State Machine               │
│  ├── Diamond (EIP-2535)      ├── Oracle                         │
│  └── Beacon                  ├── Commit-Reveal                  │
│                              └── Emergency Stop                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Factory Pattern

Паттерн Factory используется для создания новых экземпляров контрактов программно.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Контракт, который будет создаваться фабрикой
contract Token {
    string public name;
    string public symbol;
    address public owner;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;

    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        owner = msg.sender;
        totalSupply = _initialSupply;
        balanceOf[msg.sender] = _initialSupply;
    }
}

// Фабрика для создания токенов
contract TokenFactory {
    // Хранение всех созданных токенов
    address[] public deployedTokens;
    mapping(address => address[]) public tokensByCreator;

    event TokenCreated(
        address indexed tokenAddress,
        address indexed creator,
        string name,
        string symbol
    );

    // Создание нового токена
    function createToken(
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply
    ) public returns (address) {
        // Создание нового контракта
        Token newToken = new Token(_name, _symbol, _initialSupply);

        address tokenAddress = address(newToken);

        // Сохранение адреса
        deployedTokens.push(tokenAddress);
        tokensByCreator[msg.sender].push(tokenAddress);

        emit TokenCreated(tokenAddress, msg.sender, _name, _symbol);

        return tokenAddress;
    }

    // Создание с предопределённым адресом (CREATE2)
    function createTokenDeterministic(
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply,
        bytes32 _salt
    ) public returns (address) {
        Token newToken = new Token{salt: _salt}(_name, _symbol, _initialSupply);

        address tokenAddress = address(newToken);
        deployedTokens.push(tokenAddress);

        return tokenAddress;
    }

    // Предсказание адреса CREATE2
    function predictAddress(
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply,
        bytes32 _salt
    ) public view returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(Token).creationCode,
            abi.encode(_name, _symbol, _initialSupply)
        );

        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                _salt,
                keccak256(bytecode)
            )
        );

        return address(uint160(uint256(hash)));
    }

    function getDeployedTokensCount() public view returns (uint256) {
        return deployedTokens.length;
    }
}
```

## Proxy Pattern

Proxy позволяет обновлять логику контракта, сохраняя состояние и адрес.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Proxy Architecture                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     User                                                        │
│       │                                                         │
│       │ call()                                                  │
│       ▼                                                         │
│  ┌─────────────┐                                               │
│  │    Proxy    │  Storage: все данные хранятся здесь           │
│  │  Contract   │                                               │
│  └──────┬──────┘                                               │
│         │ delegatecall()                                        │
│         ▼                                                       │
│  ┌─────────────┐      ┌─────────────┐                          │
│  │Implementation│ ───▶│Implementation│  (обновление)           │
│  │     V1      │      │     V2      │                          │
│  └─────────────┘      └─────────────┘                          │
│                                                                 │
│  delegatecall выполняет код V1/V2 в контексте Proxy            │
│  (msg.sender, msg.value, storage - всё от Proxy)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Transparent Proxy

```solidity
// Storage библиотека для proxy
library StorageSlot {
    struct AddressSlot {
        address value;
    }

    function getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r.slot := slot
        }
    }
}

// Простой Transparent Proxy
contract TransparentProxy {
    // EIP-1967 стандартные слоты
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
    bytes32 private constant ADMIN_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1);

    event Upgraded(address indexed implementation);
    event AdminChanged(address previousAdmin, address newAdmin);

    constructor(address _implementation, address _admin) {
        _setImplementation(_implementation);
        _setAdmin(_admin);
    }

    modifier onlyAdmin() {
        require(msg.sender == _getAdmin(), "Not admin");
        _;
    }

    // Admin функции
    function upgradeTo(address newImplementation) external onlyAdmin {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }

    function changeAdmin(address newAdmin) external onlyAdmin {
        emit AdminChanged(_getAdmin(), newAdmin);
        _setAdmin(newAdmin);
    }

    function implementation() external view returns (address) {
        return _getImplementation();
    }

    function admin() external view returns (address) {
        return _getAdmin();
    }

    // Internal functions
    function _getImplementation() internal view returns (address) {
        return StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value;
    }

    function _setImplementation(address _impl) internal {
        require(_impl.code.length > 0, "Not a contract");
        StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value = _impl;
    }

    function _getAdmin() internal view returns (address) {
        return StorageSlot.getAddressSlot(ADMIN_SLOT).value;
    }

    function _setAdmin(address _admin) internal {
        StorageSlot.getAddressSlot(ADMIN_SLOT).value = _admin;
    }

    // Fallback для делегирования вызовов
    fallback() external payable {
        address impl = _getImplementation();
        assembly {
            // Копируем calldata
            calldatacopy(0, 0, calldatasize())

            // Делегируем вызов
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)

            // Копируем результат
            returndatacopy(0, 0, returndatasize())

            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }

    receive() external payable {}
}

// Implementation V1
contract ImplementationV1 {
    uint256 public value;
    address public owner;

    function initialize(address _owner) external {
        require(owner == address(0), "Already initialized");
        owner = _owner;
    }

    function setValue(uint256 _value) external {
        value = _value;
    }

    function getValue() external view returns (uint256) {
        return value;
    }
}

// Implementation V2 (с новой функциональностью)
contract ImplementationV2 {
    uint256 public value;
    address public owner;
    uint256 public multiplier; // Новая переменная

    function initialize(address _owner) external {
        require(owner == address(0), "Already initialized");
        owner = _owner;
    }

    function setValue(uint256 _value) external {
        value = _value;
    }

    function getValue() external view returns (uint256) {
        return value;
    }

    // Новая функция
    function setMultiplier(uint256 _multiplier) external {
        multiplier = _multiplier;
    }

    function getMultipliedValue() external view returns (uint256) {
        return value * multiplier;
    }
}
```

### UUPS Proxy (EIP-1822)

```solidity
// Base contract для UUPS
abstract contract UUPSUpgradeable {
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);

    event Upgraded(address indexed implementation);

    modifier onlyProxy() {
        require(address(this) != __self(), "Must be called through proxy");
        _;
    }

    function __self() internal view virtual returns (address);

    function _authorizeUpgrade(address newImplementation) internal virtual;

    function upgradeTo(address newImplementation) external onlyProxy {
        _authorizeUpgrade(newImplementation);
        _upgradeTo(newImplementation);
    }

    function _upgradeTo(address newImplementation) internal {
        require(newImplementation.code.length > 0, "Not a contract");
        StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value = newImplementation;
        emit Upgraded(newImplementation);
    }

    function proxiableUUID() external pure returns (bytes32) {
        return IMPLEMENTATION_SLOT;
    }
}

// Implementation с UUPS
contract MyContractV1 is UUPSUpgradeable {
    address private immutable _self;
    address public owner;
    uint256 public value;

    constructor() {
        _self = address(this);
    }

    function __self() internal view override returns (address) {
        return _self;
    }

    function initialize(address _owner) external {
        require(owner == address(0), "Already initialized");
        owner = _owner;
    }

    function _authorizeUpgrade(address) internal view override {
        require(msg.sender == owner, "Not owner");
    }

    function setValue(uint256 _value) external {
        value = _value;
    }
}
```

## Access Control Pattern

Управление доступом к функциям контракта.

```solidity
// Простой Ownable
contract Ownable {
    address public owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function renounceOwnership() public virtual onlyOwner {
        emit OwnershipTransferred(owner, address(0));
        owner = address(0);
    }
}

// Role-Based Access Control
contract AccessControl {
    struct RoleData {
        mapping(address => bool) members;
        bytes32 adminRole;
    }

    mapping(bytes32 => RoleData) private _roles;

    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    event RoleGranted(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed sender);

    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "AccessControl: missing role");
        _;
    }

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function hasRole(bytes32 role, address account) public view returns (bool) {
        return _roles[role].members[account];
    }

    function getRoleAdmin(bytes32 role) public view returns (bytes32) {
        return _roles[role].adminRole;
    }

    function grantRole(bytes32 role, address account) public onlyRole(getRoleAdmin(role)) {
        _grantRole(role, account);
    }

    function revokeRole(bytes32 role, address account) public onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }

    function renounceRole(bytes32 role) public {
        _revokeRole(role, msg.sender);
    }

    function _grantRole(bytes32 role, address account) internal {
        if (!hasRole(role, account)) {
            _roles[role].members[account] = true;
            emit RoleGranted(role, account, msg.sender);
        }
    }

    function _revokeRole(bytes32 role, address account) internal {
        if (hasRole(role, account)) {
            _roles[role].members[account] = false;
            emit RoleRevoked(role, account, msg.sender);
        }
    }

    function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal {
        _roles[role].adminRole = adminRole;
    }
}

// Пример использования
contract MyToken is AccessControl {
    mapping(address => uint256) public balanceOf;

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        balanceOf[to] += amount;
    }
}
```

## Reentrancy Guard

Защита от атак повторного входа.

```solidity
abstract contract ReentrancyGuard {
    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;

    uint256 private _status;

    error ReentrancyGuardReentrantCall();

    constructor() {
        _status = NOT_ENTERED;
    }

    modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    function _nonReentrantBefore() private {
        if (_status == ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }
        _status = ENTERED;
    }

    function _nonReentrantAfter() private {
        _status = NOT_ENTERED;
    }
}

// Пример уязвимого контракта
contract VulnerableVault {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // УЯЗВИМО! Не делайте так!
    function withdraw() external {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance");

        // Внешний вызов ДО обновления состояния
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");

        // Состояние обновляется ПОСЛЕ - уязвимость!
        balances[msg.sender] = 0;
    }
}

// Безопасный контракт с защитой
contract SecureVault is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // Паттерн Checks-Effects-Interactions
    function withdraw() external nonReentrant {
        uint256 balance = balances[msg.sender];
        require(balance > 0, "No balance"); // Checks

        balances[msg.sender] = 0; // Effects (обновление ПЕРЕД вызовом)

        (bool success, ) = msg.sender.call{value: balance}(""); // Interactions
        require(success, "Transfer failed");
    }
}
```

## Pull Payment Pattern

Безопасная модель выплат — получатели сами забирают средства.

```solidity
contract PullPayment {
    mapping(address => uint256) private _payments;

    event PaymentDeposited(address indexed payee, uint256 amount);
    event PaymentWithdrawn(address indexed payee, uint256 amount);

    // Накопление платежа для получателя
    function _asyncTransfer(address payee, uint256 amount) internal {
        _payments[payee] += amount;
        emit PaymentDeposited(payee, amount);
    }

    // Получатель сам забирает средства
    function withdrawPayments(address payable payee) public {
        uint256 payment = _payments[payee];
        require(payment > 0, "No payments");

        _payments[payee] = 0;

        (bool success, ) = payee.call{value: payment}("");
        require(success, "Transfer failed");

        emit PaymentWithdrawn(payee, payment);
    }

    function payments(address payee) public view returns (uint256) {
        return _payments[payee];
    }
}

// Пример: Аукцион с Pull Payment
contract Auction is PullPayment {
    address public highestBidder;
    uint256 public highestBid;
    bool public ended;

    event NewBid(address indexed bidder, uint256 amount);
    event AuctionEnded(address winner, uint256 amount);

    function bid() external payable {
        require(!ended, "Auction ended");
        require(msg.value > highestBid, "Bid too low");

        if (highestBidder != address(0)) {
            // Возврат предыдущей ставки через Pull Payment
            _asyncTransfer(highestBidder, highestBid);
        }

        highestBidder = msg.sender;
        highestBid = msg.value;

        emit NewBid(msg.sender, msg.value);
    }

    function endAuction() external {
        require(!ended, "Already ended");
        ended = true;
        emit AuctionEnded(highestBidder, highestBid);
    }
}
```

## State Machine Pattern

Управление жизненным циклом контракта через состояния.

```solidity
contract CrowdfundingStateMachine {
    enum State {
        Funding,    // Сбор средств
        Successful, // Цель достигнута
        Failed,     // Не удалось собрать
        Closed      // Закрыто
    }

    State public state;

    address public owner;
    uint256 public goal;
    uint256 public deadline;
    uint256 public totalFunded;

    mapping(address => uint256) public contributions;

    event StateChanged(State from, State to);
    event Contribution(address indexed contributor, uint256 amount);
    event Refund(address indexed contributor, uint256 amount);
    event FundsWithdrawn(address indexed owner, uint256 amount);

    modifier inState(State _state) {
        require(state == _state, "Invalid state");
        _;
    }

    modifier timedTransitions() {
        if (state == State.Funding && block.timestamp >= deadline) {
            _transitionState(totalFunded >= goal ? State.Successful : State.Failed);
        }
        _;
    }

    constructor(uint256 _goal, uint256 _durationDays) {
        owner = msg.sender;
        goal = _goal;
        deadline = block.timestamp + (_durationDays * 1 days);
        state = State.Funding;
    }

    function contribute() external payable inState(State.Funding) timedTransitions {
        require(block.timestamp < deadline, "Deadline passed");
        require(msg.value > 0, "Must send ETH");

        contributions[msg.sender] += msg.value;
        totalFunded += msg.value;

        emit Contribution(msg.sender, msg.value);

        // Автоматический переход при достижении цели
        if (totalFunded >= goal) {
            _transitionState(State.Successful);
        }
    }

    function claimRefund() external inState(State.Failed) {
        uint256 amount = contributions[msg.sender];
        require(amount > 0, "No contribution");

        contributions[msg.sender] = 0;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Refund failed");

        emit Refund(msg.sender, amount);
    }

    function withdrawFunds() external inState(State.Successful) {
        require(msg.sender == owner, "Not owner");

        uint256 amount = address(this).balance;
        _transitionState(State.Closed);

        (bool success, ) = owner.call{value: amount}("");
        require(success, "Withdrawal failed");

        emit FundsWithdrawn(owner, amount);
    }

    function checkState() external timedTransitions {
        // Триггер проверки состояния
    }

    function _transitionState(State newState) internal {
        emit StateChanged(state, newState);
        state = newState;
    }

    function getState() external view returns (string memory) {
        if (state == State.Funding) return "Funding";
        if (state == State.Successful) return "Successful";
        if (state == State.Failed) return "Failed";
        return "Closed";
    }
}
```

## Commit-Reveal Pattern

Защита от front-running через двухэтапное раскрытие.

```solidity
contract CommitReveal {
    struct Commitment {
        bytes32 hash;
        uint256 block;
        bool revealed;
    }

    mapping(address => Commitment) public commitments;

    uint256 public constant REVEAL_TIMEOUT = 100; // блоков

    event Committed(address indexed user, bytes32 hash);
    event Revealed(address indexed user, uint256 value, string secret);

    // Этап 1: Commit (отправка хеша)
    function commit(bytes32 _hash) external {
        require(commitments[msg.sender].hash == 0, "Already committed");

        commitments[msg.sender] = Commitment({
            hash: _hash,
            block: block.number,
            revealed: false
        });

        emit Committed(msg.sender, _hash);
    }

    // Этап 2: Reveal (раскрытие значения)
    function reveal(uint256 _value, string memory _secret) external {
        Commitment storage commitment = commitments[msg.sender];

        require(commitment.hash != 0, "No commitment");
        require(!commitment.revealed, "Already revealed");
        require(block.number > commitment.block, "Too early");
        require(block.number <= commitment.block + REVEAL_TIMEOUT, "Timeout");

        // Проверка хеша
        bytes32 calculatedHash = keccak256(abi.encodePacked(_value, _secret, msg.sender));
        require(calculatedHash == commitment.hash, "Invalid reveal");

        commitment.revealed = true;

        emit Revealed(msg.sender, _value, _secret);

        // Здесь можно использовать _value для логики
    }

    // Вычисление хеша для commit (вызывать off-chain)
    function calculateHash(
        uint256 _value,
        string memory _secret,
        address _user
    ) external pure returns (bytes32) {
        return keccak256(abi.encodePacked(_value, _secret, _user));
    }
}

// Пример: Игра в угадывание числа
contract GuessTheNumber is CommitReveal {
    uint256 public targetNumber;
    address public winner;
    uint256 public prize;

    event GameStarted(uint256 prize);
    event WinnerDeclared(address winner, uint256 prize);

    function startGame(uint256 _target) external payable {
        require(winner == address(0), "Game in progress");
        targetNumber = _target;
        prize = msg.value;
        emit GameStarted(prize);
    }

    function checkWin(uint256 _guess) external {
        Commitment storage commitment = commitments[msg.sender];
        require(commitment.revealed, "Must reveal first");

        if (_guess == targetNumber && winner == address(0)) {
            winner = msg.sender;

            (bool success, ) = winner.call{value: prize}("");
            require(success, "Prize transfer failed");

            emit WinnerDeclared(winner, prize);
        }
    }
}
```

## Emergency Stop (Circuit Breaker)

Паттерн для экстренной остановки контракта.

```solidity
contract Pausable {
    bool public paused;
    address public guardian;

    event Paused(address account);
    event Unpaused(address account);

    modifier whenNotPaused() {
        require(!paused, "Pausable: paused");
        _;
    }

    modifier whenPaused() {
        require(paused, "Pausable: not paused");
        _;
    }

    modifier onlyGuardian() {
        require(msg.sender == guardian, "Not guardian");
        _;
    }

    constructor() {
        guardian = msg.sender;
    }

    function pause() external onlyGuardian whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }

    function unpause() external onlyGuardian whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
}

// Расширенный Circuit Breaker с таймлоком
contract EmergencyStop {
    bool public stopped;
    address public owner;
    uint256 public lastStopTime;
    uint256 public constant STOP_DURATION = 1 days;

    event EmergencyStopped(address by, uint256 until);
    event Resumed(address by);

    modifier stoppable() {
        require(!stopped || block.timestamp > lastStopTime + STOP_DURATION, "Stopped");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function emergencyStop() external onlyOwner {
        stopped = true;
        lastStopTime = block.timestamp;
        emit EmergencyStopped(msg.sender, block.timestamp + STOP_DURATION);
    }

    function resume() external onlyOwner {
        require(stopped, "Not stopped");
        stopped = false;
        emit Resumed(msg.sender);
    }

    // Автоматическое возобновление после таймаута
    function checkResume() external {
        if (stopped && block.timestamp > lastStopTime + STOP_DURATION) {
            stopped = false;
            emit Resumed(address(0));
        }
    }
}
```

## Minimal Proxy (Clone Pattern)

Создание дешёвых клонов контракта с использованием EIP-1167.

```solidity
contract MinimalProxyFactory {
    address public implementation;

    event ProxyCreated(address proxy);

    constructor(address _implementation) {
        implementation = _implementation;
    }

    function clone() external returns (address instance) {
        // EIP-1167 минимальный прокси байткод
        bytes20 targetBytes = bytes20(implementation);

        assembly {
            let ptr := mload(0x40)

            // 3d602d80600a3d3981f3363d3d373d3d3d363d73 + address + 5af43d82803e903d91602b57fd5bf3
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
            mstore(add(ptr, 0x14), targetBytes)
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)

            instance := create(0, ptr, 0x37)
        }

        require(instance != address(0), "Clone failed");
        emit ProxyCreated(instance);
    }

    function cloneDeterministic(bytes32 salt) external returns (address instance) {
        bytes20 targetBytes = bytes20(implementation);

        assembly {
            let ptr := mload(0x40)

            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
            mstore(add(ptr, 0x14), targetBytes)
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)

            instance := create2(0, ptr, 0x37, salt)
        }

        require(instance != address(0), "Clone failed");
        emit ProxyCreated(instance);
    }

    function predictDeterministicAddress(bytes32 salt) external view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(_cloneBytecode())
            )
        );
        return address(uint160(uint256(hash)));
    }

    function _cloneBytecode() internal view returns (bytes memory) {
        return abi.encodePacked(
            hex"3d602d80600a3d3981f3363d3d373d3d3d363d73",
            implementation,
            hex"5af43d82803e903d91602b57fd5bf3"
        );
    }
}
```

## Best Practices

### Таблица применения паттернов

| Паттерн | Когда использовать | Gas Cost |
|---------|-------------------|----------|
| Factory | Создание множества однотипных контрактов | Высокий |
| Minimal Proxy | Массовое создание дешёвых клонов | Низкий |
| Transparent Proxy | Обновляемые контракты (admin-oriented) | Средний |
| UUPS Proxy | Обновляемые контракты (gas-efficient) | Средний |
| Access Control | Многоуровневые права доступа | Низкий |
| Reentrancy Guard | Защита от атак | Минимальный |
| Pull Payment | Безопасные выплаты | Низкий |
| State Machine | Сложные workflow | Низкий |
| Commit-Reveal | Защита от front-running | Средний |
| Emergency Stop | Критические системы | Минимальный |

### Типичные ошибки

```solidity
contract CommonMistakes {
    // Ошибка 1: Инициализация proxy через constructor
    // constructor() { owner = msg.sender; } // Не работает в proxy!

    // Правильно: использовать initialize()
    bool private initialized;
    address public owner;

    function initialize(address _owner) external {
        require(!initialized, "Already initialized");
        initialized = true;
        owner = _owner;
    }

    // Ошибка 2: Storage collision в proxy
    // Слоты должны совпадать между версиями!

    // Ошибка 3: Забыть nonReentrant
    // Всегда использовать для функций с external calls

    // Ошибка 4: Push вместо Pull для выплат
    // Используйте Pull Payment для множественных выплат
}
```

## Заключение

Паттерны проектирования в Solidity помогают создавать безопасные, эффективные и поддерживаемые смарт-контракты. Выбор паттерна зависит от конкретных требований: нужна ли обновляемость, важна ли экономия газа, какой уровень безопасности требуется. Комбинирование нескольких паттернов позволяет создавать сложные децентрализованные приложения.

---

**Навигация:**
- [Обзор Solidity](../readme.md)
- [Основы Solidity](../01-basics/readme.md)
- [Продвинутый Solidity](../02-advanced/readme.md)
