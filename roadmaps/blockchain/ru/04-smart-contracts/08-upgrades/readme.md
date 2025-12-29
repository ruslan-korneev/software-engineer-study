# Обновление смарт-контрактов

## Проблема неизменяемости контрактов

Смарт-контракты по своей природе **неизменяемы** (immutable). После деплоя на блокчейн код контракта нельзя изменить. Это создаёт серьёзные проблемы:

```
┌─────────────────────────────────────────────────────────────┐
│                  Проблемы неизменяемости                    │
├─────────────────────────────────────────────────────────────┤
│  1. Исправление багов невозможно после деплоя               │
│  2. Добавление новых функций требует нового контракта       │
│  3. Миграция данных и пользователей сложна и дорога         │
│  4. Обнаруженные уязвимости остаются навсегда               │
└─────────────────────────────────────────────────────────────┘
```

Для решения этих проблем разработаны **паттерны обновления** (upgrade patterns).

---

## Паттерны обновления

### Proxy Pattern

Основная идея: разделить **логику** и **данные** на разные контракты.

```
┌──────────────────────────────────────────────────────────────────┐
│                      PROXY PATTERN                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Пользователь                                                    │
│       │                                                           │
│       ▼                                                           │
│   ┌─────────────┐    delegatecall    ┌─────────────────────┐     │
│   │    Proxy    │ ─────────────────► │   Implementation    │     │
│   │  (Storage)  │                    │      (Logic)        │     │
│   │             │                    │                     │     │
│   │  - owner    │                    │  - функции()        │     │
│   │  - balance  │                    │  - бизнес-логика    │     │
│   │  - impl_adr │                    │                     │     │
│   └─────────────┘                    └─────────────────────┘     │
│         │                                                         │
│         │  upgrade                                                │
│         ▼                                                         │
│   ┌─────────────────────┐                                        │
│   │  Implementation V2  │                                        │
│   │    (New Logic)      │                                        │
│   └─────────────────────┘                                        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Ключевой механизм — `delegatecall`:**
- Код выполняется из Implementation контракта
- Но данные (storage) читаются/записываются в Proxy контракт
- Адрес Proxy остаётся неизменным для пользователей

### Transparent Proxy Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TransparentProxy {
    // Слоты хранения по EIP-1967
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
    bytes32 private constant ADMIN_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1);

    constructor(address _implementation, address _admin) {
        _setImplementation(_implementation);
        _setAdmin(_admin);
    }

    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }

    function upgradeTo(address newImplementation) external ifAdmin {
        _setImplementation(newImplementation);
    }

    function _fallback() internal {
        address impl = _getImplementation();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    fallback() external payable {
        _fallback();
    }

    receive() external payable {
        _fallback();
    }

    function _getImplementation() internal view returns (address impl) {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            impl := sload(slot)
        }
    }

    function _setImplementation(address newImpl) internal {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            sstore(slot, newImpl)
        }
    }

    function _getAdmin() internal view returns (address admin) {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            admin := sload(slot)
        }
    }

    function _setAdmin(address newAdmin) internal {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            sstore(slot, newAdmin)
        }
    }
}
```

**Особенности Transparent Proxy:**
- Admin может только управлять (upgrade), не может вызывать функции implementation
- Пользователи могут только вызывать функции, не могут управлять
- Защита от коллизий селекторов функций

### UUPS Pattern (Universal Upgradeable Proxy Standard)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// Минимальный UUPS Proxy
contract UUPSProxy {
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);

    constructor(address _implementation, bytes memory _data) {
        _setImplementation(_implementation);
        if (_data.length > 0) {
            (bool success,) = _implementation.delegatecall(_data);
            require(success, "Initialization failed");
        }
    }

    fallback() external payable {
        address impl = _getImplementation();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    function _getImplementation() internal view returns (address impl) {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            impl := sload(slot)
        }
    }

    function _setImplementation(address newImpl) internal {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            sstore(slot, newImpl)
        }
    }
}

// Implementation с логикой обновления
abstract contract UUPSUpgradeable {
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);

    function upgradeTo(address newImplementation) external virtual {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCall(newImplementation, "");
    }

    function _authorizeUpgrade(address newImplementation) internal virtual;

    function _upgradeToAndCall(address newImpl, bytes memory data) internal {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            sstore(slot, newImpl)
        }
        if (data.length > 0) {
            (bool success,) = newImpl.delegatecall(data);
            require(success);
        }
    }
}
```

**Сравнение Transparent vs UUPS:**

```
┌────────────────────┬─────────────────────┬─────────────────────┐
│     Критерий       │  Transparent Proxy  │     UUPS Proxy      │
├────────────────────┼─────────────────────┼─────────────────────┤
│ Логика обновления  │     В Proxy         │  В Implementation   │
│ Газ на деплой      │     Выше            │     Ниже            │
│ Газ на вызовы      │     Выше            │     Ниже            │
│ Риск потери upgrade│     Нет             │     Да (если забыть)│
│ Размер Proxy       │     Больше          │     Меньше          │
└────────────────────┴─────────────────────┴─────────────────────┘
```

### Diamond Pattern (EIP-2535)

Diamond позволяет иметь **несколько implementation контрактов** (facets):

```
┌──────────────────────────────────────────────────────────────────┐
│                      DIAMOND PATTERN                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│                        ┌─────────────┐                           │
│                        │   Diamond   │                           │
│                        │   (Proxy)   │                           │
│                        └──────┬──────┘                           │
│                               │                                   │
│         ┌─────────────────────┼─────────────────────┐            │
│         │                     │                     │            │
│         ▼                     ▼                     ▼            │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐       │
│   │  Facet A  │        │  Facet B  │        │  Facet C  │       │
│   │ (ERC20)   │        │ (Staking) │        │  (Admin)  │       │
│   └───────────┘        └───────────┘        └───────────┘       │
│                                                                   │
│   Каждый facet отвечает за свой набор функций                    │
│   Можно обновлять facets независимо друг от друга                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

struct FacetCut {
    address facetAddress;
    uint8 action; // 0=Add, 1=Replace, 2=Remove
    bytes4[] functionSelectors;
}

contract Diamond {
    // Маппинг: selector => facet address
    mapping(bytes4 => address) public selectorToFacet;

    function diamondCut(FacetCut[] calldata cuts) external {
        // Проверка прав доступа
        require(msg.sender == owner(), "Not owner");

        for (uint i = 0; i < cuts.length; i++) {
            FacetCut memory cut = cuts[i];
            for (uint j = 0; j < cut.functionSelectors.length; j++) {
                bytes4 selector = cut.functionSelectors[j];
                if (cut.action == 0) { // Add
                    selectorToFacet[selector] = cut.facetAddress;
                } else if (cut.action == 1) { // Replace
                    selectorToFacet[selector] = cut.facetAddress;
                } else if (cut.action == 2) { // Remove
                    delete selectorToFacet[selector];
                }
            }
        }
    }

    fallback() external payable {
        address facet = selectorToFacet[msg.sig];
        require(facet != address(0), "Function not found");

        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    function owner() internal view returns (address) {
        // Реализация получения owner
    }
}
```

### Eternal Storage Pattern

Паттерн для максимальной гибкости хранения данных:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EternalStorage {
    mapping(bytes32 => uint256) internal uintStorage;
    mapping(bytes32 => string) internal stringStorage;
    mapping(bytes32 => address) internal addressStorage;
    mapping(bytes32 => bool) internal boolStorage;
    mapping(bytes32 => bytes) internal bytesStorage;

    // Геттеры
    function getUint(bytes32 key) public view returns (uint256) {
        return uintStorage[key];
    }

    function getString(bytes32 key) public view returns (string memory) {
        return stringStorage[key];
    }

    // Сеттеры (только для авторизованных контрактов)
    function setUint(bytes32 key, uint256 value) public onlyAuthorized {
        uintStorage[key] = value;
    }

    function setString(bytes32 key, string memory value) public onlyAuthorized {
        stringStorage[key] = value;
    }

    modifier onlyAuthorized() {
        require(authorizedContracts[msg.sender], "Not authorized");
        _;
    }

    mapping(address => bool) public authorizedContracts;
}

// Использование
contract MyLogicV1 {
    EternalStorage public store;

    function setUserBalance(address user, uint256 balance) external {
        bytes32 key = keccak256(abi.encodePacked("balance", user));
        store.setUint(key, balance);
    }
}
```

---

## OpenZeppelin Upgrades

OpenZeppelin предоставляет готовые решения для обновляемых контрактов:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyContractV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(uint256 _value) public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
        value = _value;
    }

    function setValue(uint256 _value) external {
        value = _value;
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}

contract MyContractV2 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;
    uint256 public newValue; // Новая переменная в V2

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(uint256 _value) public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
        value = _value;
    }

    function setValue(uint256 _value) external {
        value = _value;
    }

    function setNewValue(uint256 _newValue) external {
        newValue = _newValue;
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```

---

## Storage Layout и совместимость

**КРИТИЧЕСКИ ВАЖНО**: При обновлении контракта нельзя менять порядок существующих переменных!

```
┌──────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYOUT RULES                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   V1 Storage:              V2 Storage (ПРАВИЛЬНО):               │
│   ┌─────────────────┐      ┌─────────────────┐                   │
│   │ slot 0: owner   │      │ slot 0: owner   │  ◄── Тот же слот  │
│   │ slot 1: balance │      │ slot 1: balance │  ◄── Тот же слот  │
│   │ slot 2: name    │      │ slot 2: name    │  ◄── Тот же слот  │
│   └─────────────────┘      │ slot 3: newVar  │  ◄── НОВЫЙ слот   │
│                            └─────────────────┘                   │
│                                                                   │
│   V2 Storage (НЕПРАВИЛЬНО - сломает данные!):                    │
│   ┌─────────────────┐                                            │
│   │ slot 0: newVar  │  ◄── ОШИБКА! Перезапишет owner            │
│   │ slot 1: owner   │                                            │
│   │ slot 2: balance │                                            │
│   └─────────────────┘                                            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Правила совместимости storage:**

```solidity
// V1
contract MyContractV1 {
    uint256 public value;      // slot 0
    address public owner;      // slot 1
    mapping(address => uint256) public balances; // slot 2
}

// V2 - ПРАВИЛЬНО
contract MyContractV2 {
    uint256 public value;      // slot 0 - без изменений
    address public owner;      // slot 1 - без изменений
    mapping(address => uint256) public balances; // slot 2 - без изменений
    uint256 public newValue;   // slot 3 - НОВАЯ переменная в конце
    string public name;        // slot 4 - НОВАЯ переменная в конце
}

// V2 - НЕПРАВИЛЬНО (НЕ ДЕЛАЙТЕ ТАК!)
contract MyContractV2Bad {
    uint256 public newValue;   // ОШИБКА: изменён порядок
    uint256 public value;
    address public owner;
    mapping(address => uint256) public balances;
}
```

---

## Риски и лучшие практики обновлений

### Риски

1. **Storage Collision** — коллизия слотов хранения
2. **Function Selector Clash** — одинаковые селекторы в proxy и implementation
3. **Initialization Attack** — повторный вызов initialize()
4. **Uninitialized Implementation** — атака через неинициализированный implementation

### Лучшие практики

```solidity
// 1. Всегда отключайте initializers в конструкторе implementation
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}

// 2. Используйте storage gaps для будущих переменных
contract MyContractV1 {
    uint256 public value;

    // Резервируем 50 слотов для будущих переменных
    uint256[50] private __gap;
}

// 3. Используйте reinitializer для обновления initializer
function initializeV2(uint256 _newValue) public reinitializer(2) {
    newValue = _newValue;
}

// 4. Проверяйте совместимость с помощью OpenZeppelin Upgrades плагина
// npx hardhat run scripts/upgrade.js
```

---

## Governance и Multisig для обновлений

Для безопасного управления обновлениями используйте multisig и timelock:

```
┌──────────────────────────────────────────────────────────────────┐
│              GOVERNANCE UPGRADE FLOW                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   1. Propose      2. Queue         3. Wait         4. Execute    │
│   ┌─────────┐    ┌─────────┐     ┌─────────┐     ┌─────────┐    │
│   │ Multisig│───►│Timelock │───► │ 48h     │───► │ Upgrade │    │
│   │ (3/5)   │    │         │     │ delay   │     │         │    │
│   └─────────┘    └─────────┘     └─────────┘     └─────────┘    │
│                                                                   │
│   Пользователи могут выйти из протокола до применения обновления │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/TimelockController.sol";

contract UpgradeableWithTimelock is UUPSUpgradeable {
    TimelockController public timelock;

    function _authorizeUpgrade(address newImplementation)
        internal
        override
    {
        // Только timelock может обновлять контракт
        require(
            msg.sender == address(timelock),
            "Only timelock can upgrade"
        );
    }
}

// Пример Multisig + Timelock setup
contract GovernanceSetup {
    function setupGovernance() external {
        address[] memory proposers = new address[](1);
        proposers[0] = multisigAddress;

        address[] memory executors = new address[](1);
        executors[0] = multisigAddress;

        // Timelock с 48-часовой задержкой
        TimelockController timelock = new TimelockController(
            48 hours,    // minDelay
            proposers,   // proposers
            executors,   // executors
            address(0)   // admin (отключён)
        );
    }
}
```

---

## Чеклист обновления контракта

```
┌──────────────────────────────────────────────────────────────────┐
│              UPGRADE CHECKLIST                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [ ] Storage layout совместим с предыдущей версией               │
│  [ ] Новые переменные добавлены ТОЛЬКО в конец                   │
│  [ ] Типы существующих переменных НЕ изменены                    │
│  [ ] Тесты на storage layout пройдены                            │
│  [ ] Аудит нового кода выполнен                                  │
│  [ ] Timelock задержка достаточная (48h+)                        │
│  [ ] Multisig кворум настроен (3/5 или выше)                     │
│  [ ] Пользователи уведомлены о предстоящем обновлении            │
│  [ ] Откат (rollback) план подготовлен                           │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Дополнительные ресурсы

- [OpenZeppelin Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins)
- [EIP-1967: Standard Proxy Storage Slots](https://eips.ethereum.org/EIPS/eip-1967)
- [EIP-2535: Diamond Standard](https://eips.ethereum.org/EIPS/eip-2535)
- [OpenZeppelin Proxy Contracts](https://docs.openzeppelin.com/contracts/4.x/api/proxy)
