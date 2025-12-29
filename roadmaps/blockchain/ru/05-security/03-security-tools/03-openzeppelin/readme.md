# OpenZeppelin Contracts

OpenZeppelin Contracts - это библиотека безопасных, проверенных и многократно аудированных смарт-контрактов для Ethereum и EVM-совместимых блокчейнов.

## Почему OpenZeppelin

- **Безопасность**: Код прошел множество аудитов от ведущих компаний
- **Стандартизация**: Реализации соответствуют EIP стандартам
- **Тестирование**: 100% покрытие тестами
- **Сообщество**: Огромное community и поддержка
- **Обновления**: Регулярные security patches

## Установка

### Foundry

```bash
# Установка через forge
forge install OpenZeppelin/openzeppelin-contracts

# Добавить remapping в foundry.toml
# [profile.default]
# remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/"]
```

### Hardhat / npm

```bash
npm install @openzeppelin/contracts
# или
yarn add @openzeppelin/contracts
```

### Версии

```bash
# Последняя стабильная версия (рекомендуется)
npm install @openzeppelin/contracts@5.0.0

# Для старых проектов
npm install @openzeppelin/contracts@4.9.3
```

## Структура библиотеки

```
@openzeppelin/contracts/
├── access/           # Access control (Ownable, AccessControl, etc.)
├── finance/          # Financial primitives (PaymentSplitter, VestingWallet)
├── governance/       # DAO и voting
├── interfaces/       # Стандартные интерфейсы
├── metatx/           # Meta-transactions
├── proxy/            # Proxy patterns для upgrades
├── security/         # ReentrancyGuard, Pausable
├── token/
│   ├── ERC20/       # Fungible tokens
│   ├── ERC721/      # NFTs
│   ├── ERC1155/     # Multi-tokens
│   └── common/      # Общие утилиты
└── utils/            # Helpers (Address, Strings, Math, etc.)
```

## Token Standards

### ERC20 - Fungible Tokens

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import {ERC20Pausable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, ERC20Burnable, ERC20Pausable, ERC20Permit, Ownable {
    constructor(address initialOwner)
        ERC20("MyToken", "MTK")
        ERC20Permit("MyToken")
        Ownable(initialOwner)
    {
        _mint(msg.sender, 1000000 * 10 ** decimals());
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    // Required override для ERC20Pausable
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Pausable)
    {
        super._update(from, to, value);
    }
}
```

### ERC20 Extensions

```solidity
// ERC20Capped - ограничение общего supply
import {ERC20Capped} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Capped.sol";

contract CappedToken is ERC20Capped {
    constructor() ERC20("Capped", "CAP") ERC20Capped(1000000 * 10**18) {}
}

// ERC20Votes - governance токен с voting power
import {ERC20Votes} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import {Nonces} from "@openzeppelin/contracts/utils/Nonces.sol";

contract GovernanceToken is ERC20, ERC20Permit, ERC20Votes {
    constructor() ERC20("Gov", "GOV") ERC20Permit("Gov") {}

    // Overrides для корректной работы
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Votes)
    {
        super._update(from, to, value);
    }

    function nonces(address owner)
        public
        view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
}

// ERC20Snapshot - snapshots балансов
import {ERC20Snapshot} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Snapshot.sol";

contract SnapshotToken is ERC20Snapshot {
    function snapshot() public returns (uint256) {
        return _snapshot();
    }

    function balanceOfAt(address account, uint256 snapshotId)
        public view returns (uint256)
    {
        return super.balanceOfAt(account, snapshotId);
    }
}
```

### ERC721 - Non-Fungible Tokens

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {ERC721Enumerable} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import {ERC721URIStorage} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import {ERC721Pausable} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Pausable.sol";
import {ERC721Burnable} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721Burnable.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract MyNFT is
    ERC721,
    ERC721Enumerable,
    ERC721URIStorage,
    ERC721Pausable,
    ERC721Burnable,
    AccessControl
{
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    uint256 private _nextTokenId;

    constructor(address defaultAdmin, address minter)
        ERC721("MyNFT", "MNFT")
    {
        _grantRole(DEFAULT_ADMIN_ROLE, defaultAdmin);
        _grantRole(MINTER_ROLE, minter);
    }

    function pause() public onlyRole(DEFAULT_ADMIN_ROLE) {
        _pause();
    }

    function unpause() public onlyRole(DEFAULT_ADMIN_ROLE) {
        _unpause();
    }

    function safeMint(address to, string memory uri) public onlyRole(MINTER_ROLE) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
    }

    // Required overrides
    function _update(address to, uint256 tokenId, address auth)
        internal
        override(ERC721, ERC721Enumerable, ERC721Pausable)
        returns (address)
    {
        return super._update(to, tokenId, auth);
    }

    function _increaseBalance(address account, uint128 value)
        internal
        override(ERC721, ERC721Enumerable)
    {
        super._increaseBalance(account, value);
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (string memory)
    {
        return super.tokenURI(tokenId);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721Enumerable, ERC721URIStorage, AccessControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

### ERC1155 - Multi-Token Standard

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC1155} from "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import {ERC1155Burnable} from "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import {ERC1155Supply} from "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract GameItems is ERC1155, ERC1155Burnable, ERC1155Supply, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    // Item IDs
    uint256 public constant GOLD = 0;
    uint256 public constant SILVER = 1;
    uint256 public constant SWORD = 2;
    uint256 public constant SHIELD = 3;

    constructor(address defaultAdmin, address minter)
        ERC1155("https://game.example/api/item/{id}.json")
    {
        _grantRole(DEFAULT_ADMIN_ROLE, defaultAdmin);
        _grantRole(MINTER_ROLE, minter);
    }

    function setURI(string memory newuri) public onlyRole(DEFAULT_ADMIN_ROLE) {
        _setURI(newuri);
    }

    function mint(
        address account,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public onlyRole(MINTER_ROLE) {
        _mint(account, id, amount, data);
    }

    function mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public onlyRole(MINTER_ROLE) {
        _mintBatch(to, ids, amounts, data);
    }

    // Required overrides
    function _update(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values
    ) internal override(ERC1155, ERC1155Supply) {
        super._update(from, to, ids, values);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC1155, AccessControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

## Access Control

### Ownable

Простейший паттерн управления доступом:

```solidity
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
    constructor(address initialOwner) Ownable(initialOwner) {}

    function privilegedAction() public onlyOwner {
        // Только owner может вызвать
    }

    // Встроенные функции:
    // - owner() - возвращает текущего owner
    // - transferOwnership(address) - передача ownership
    // - renounceOwnership() - отказ от ownership
}
```

### Ownable2Step

Двухэтапная передача ownership для безопасности:

```solidity
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract MyContract is Ownable2Step {
    constructor(address initialOwner) Ownable(initialOwner) {}

    // transferOwnership() устанавливает pendingOwner
    // pendingOwner должен вызвать acceptOwnership()
}
```

### AccessControl

Гибкая система ролей:

```solidity
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract Treasury is AccessControl {
    bytes32 public constant TREASURER_ROLE = keccak256("TREASURER_ROLE");
    bytes32 public constant AUDITOR_ROLE = keccak256("AUDITOR_ROLE");

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
    }

    function withdraw(uint256 amount) public onlyRole(TREASURER_ROLE) {
        // Только treasurer может снимать
    }

    function audit() public onlyRole(AUDITOR_ROLE) {
        // Только auditor может аудитировать
    }

    // Admin может:
    // - grantRole(role, account)
    // - revokeRole(role, account)
    // - Пользователь может: renounceRole(role, account)
}
```

### AccessControlEnumerable

AccessControl с возможностью перечисления членов роли:

```solidity
import {AccessControlEnumerable} from "@openzeppelin/contracts/access/extensions/AccessControlEnumerable.sol";

contract MyContract is AccessControlEnumerable {
    // Дополнительные функции:
    // - getRoleMemberCount(role)
    // - getRoleMember(role, index)
}
```

## Security Utilities

### ReentrancyGuard

Защита от reentrancy атак:

```solidity
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // nonReentrant предотвращает повторный вход
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### Pausable

Механизм паузы контракта:

```solidity
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Pausable, Ownable {
    constructor(address initialOwner) Ownable(initialOwner) {}

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function transfer(address to, uint256 amount) public whenNotPaused {
        // Работает только когда не на паузе
    }

    function emergencyWithdraw() public whenPaused {
        // Работает только на паузе
    }
}
```

### PullPayment

Безопасные выплаты через pull pattern:

```solidity
import {PullPayment} from "@openzeppelin/contracts/security/PullPayment.sol";

contract Auction is PullPayment {
    address public highestBidder;
    uint256 public highestBid;

    function bid() external payable {
        require(msg.value > highestBid, "Bid too low");

        if (highestBidder != address(0)) {
            // Вместо прямой отправки - записываем для withdraw
            _asyncTransfer(highestBidder, highestBid);
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    // Пользователи вызывают withdrawPayments() для получения средств
}
```

## Utilities

### Address

```solidity
import {Address} from "@openzeppelin/contracts/utils/Address.sol";

contract MyContract {
    using Address for address;
    using Address for address payable;

    function example(address target) external {
        // Проверка, является ли адрес контрактом
        require(target.code.length > 0, "Not a contract");

        // Безопасный вызов с проверкой результата
        bytes memory data = target.functionCall(
            abi.encodeWithSignature("someFunction()")
        );

        // Отправка ETH с проверкой
        payable(target).sendValue(1 ether);
    }
}
```

### Strings

```solidity
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";

contract MyNFT {
    using Strings for uint256;
    using Strings for address;

    function tokenURI(uint256 tokenId) public pure returns (string memory) {
        return string.concat(
            "https://example.com/token/",
            tokenId.toString(),
            ".json"
        );
    }

    function addressToString(address account) public pure returns (string memory) {
        return account.toHexString();
    }
}
```

### Math

```solidity
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {SignedMath} from "@openzeppelin/contracts/utils/math/SignedMath.sol";

contract Calculator {
    using Math for uint256;

    function example() public pure {
        // Максимум/минимум
        uint256 max = Math.max(100, 200);  // 200
        uint256 min = Math.min(100, 200);  // 100

        // Среднее без overflow
        uint256 avg = Math.average(type(uint256).max, 100);

        // Деление с округлением вверх
        uint256 ceilDiv = Math.ceilDiv(10, 3);  // 4

        // Квадратный корень
        uint256 sqrt = Math.sqrt(16);  // 4

        // Логарифм
        uint256 log2 = Math.log2(8);   // 3
        uint256 log10 = Math.log10(100);  // 2
    }
}
```

### Arrays

```solidity
import {Arrays} from "@openzeppelin/contracts/utils/Arrays.sol";

contract MyContract {
    using Arrays for uint256[];

    uint256[] public values;

    function findValue(uint256 value) public view returns (uint256) {
        // Бинарный поиск (массив должен быть отсортирован)
        return values.findUpperBound(value);
    }
}
```

### Counters (deprecated в v5, используйте обычный uint256)

```solidity
// В OpenZeppelin v5 просто используйте:
contract MyContract {
    uint256 private _tokenIdCounter;

    function mint() public {
        uint256 tokenId = _tokenIdCounter;
        _tokenIdCounter++;
        // mint logic
    }
}
```

## Upgradeable Contracts

OpenZeppelin предоставляет отдельный пакет для upgradeable контрактов.

### Установка

```bash
# Foundry
forge install OpenZeppelin/openzeppelin-contracts-upgradeable

# npm
npm install @openzeppelin/contracts-upgradeable
```

### Базовый Upgradeable контракт

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {ERC20Upgradeable} from "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MyTokenV1 is Initializable, ERC20Upgradeable, OwnableUpgradeable, UUPSUpgradeable {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address initialOwner) public initializer {
        __ERC20_init("MyToken", "MTK");
        __Ownable_init(initialOwner);
        __UUPSUpgradeable_init();

        _mint(msg.sender, 1000000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```

### Важные правила Upgradeable контрактов

```solidity
// 1. НЕ используйте конструктор для инициализации
// BAD:
constructor() {
    owner = msg.sender;  // Не работает!
}

// GOOD:
function initialize(address _owner) public initializer {
    owner = _owner;
}

// 2. НЕ инициализируйте переменные при объявлении
// BAD:
uint256 public value = 100;  // Не работает!

// GOOD:
uint256 public value;
function initialize() public initializer {
    value = 100;
}

// 3. Используйте storage gaps для будущих переменных
contract MyContractV1 {
    uint256 public value;

    // Резервируем слоты для будущих версий
    uint256[49] private __gap;
}

contract MyContractV2 {
    uint256 public value;
    uint256 public newValue;  // Использует один слот из __gap

    uint256[48] private __gap;  // Уменьшаем на 1
}
```

### Proxy Patterns

```solidity
// UUPS (Universal Upgradeable Proxy Standard)
// Логика upgrade в implementation
import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

// Transparent Proxy
// Логика upgrade в proxy
import {TransparentUpgradeableProxy} from "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Beacon Proxy
// Для множества прокси с одной implementation
import {BeaconProxy} from "@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol";
```

### Деплой Upgradeable контракта (Foundry)

```solidity
// script/DeployUpgradeable.s.sol
pragma solidity ^0.8.20;

import {Script} from "forge-std/Script.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {MyTokenV1} from "../src/MyTokenV1.sol";

contract DeployScript is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);

        // Deploy implementation
        MyTokenV1 implementation = new MyTokenV1();

        // Deploy proxy
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(implementation),
            abi.encodeCall(MyTokenV1.initialize, (msg.sender))
        );

        // Теперь взаимодействуем с proxy как с MyTokenV1
        MyTokenV1 token = MyTokenV1(address(proxy));

        vm.stopBroadcast();
    }
}
```

## OpenZeppelin Defender

OpenZeppelin Defender - это платформа для безопасного управления смарт-контрактами.

### Возможности

- **Admin**: Безопасное управление контрактами через multisig
- **Autotask**: Автоматизация действий (keeper functions)
- **Sentinel**: Мониторинг транзакций и событий
- **Relay**: Gasless транзакции

### Интеграция с Defender

```typescript
// defender.config.ts
import { DefenderRelaySigner } from '@openzeppelin/defender-relay-client/lib/ethers';

const credentials = {
  apiKey: process.env.DEFENDER_API_KEY,
  apiSecret: process.env.DEFENDER_API_SECRET,
};

const provider = new DefenderRelayProvider(credentials);
const signer = new DefenderRelaySigner(credentials, provider);

// Использование signer для транзакций
const contract = new Contract(address, abi, signer);
await contract.someFunction();
```

## Best Practices

### 1. Всегда используйте последнюю стабильную версию

```bash
npm install @openzeppelin/contracts@latest
```

### 2. Читайте документацию для каждого контракта

```solidity
// Каждый контракт имеет NatSpec документацию
/// @notice Transfers `amount` tokens from the caller to `to`
/// @param to The address to transfer to
/// @param amount The amount to transfer
/// @return True if the transfer succeeded
function transfer(address to, uint256 amount) public returns (bool);
```

### 3. Переопределяйте функции правильно

```solidity
// Используйте override для всех перегружаемых функций
function _update(address from, address to, uint256 amount)
    internal
    override(ERC20, ERC20Pausable)  // Указываем все parent контракты
{
    super._update(from, to, amount);  // Вызываем parent
}
```

### 4. Тестируйте интеграцию

```solidity
// test/MyToken.t.sol
import {Test} from "forge-std/Test.sol";
import {MyToken} from "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken public token;

    function setUp() public {
        token = new MyToken(address(this));
    }

    function testMint() public {
        token.mint(address(1), 100);
        assertEq(token.balanceOf(address(1)), 100);
    }

    function testPause() public {
        token.pause();
        vm.expectRevert();
        token.transfer(address(1), 100);
    }
}
```

### 5. Используйте Wizard для генерации кода

OpenZeppelin Contracts Wizard: https://wizard.openzeppelin.com/

Генерирует код контрактов с нужными extensions.

## Ресурсы

- [Официальная документация](https://docs.openzeppelin.com/contracts/5.x/)
- [GitHub репозиторий](https://github.com/OpenZeppelin/openzeppelin-contracts)
- [Contracts Wizard](https://wizard.openzeppelin.com/)
- [Security Audits](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/audits)
- [OpenZeppelin Forum](https://forum.openzeppelin.com/)
- [Defender Platform](https://defender.openzeppelin.com/)
