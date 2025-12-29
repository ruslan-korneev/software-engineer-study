# Remix IDE

## Что такое Remix IDE

**Remix IDE** — это браузерная интегрированная среда разработки для создания, тестирования и деплоя смарт-контрактов на Ethereum и других EVM-совместимых блокчейнах. Remix разработан командой Ethereum Foundation и является одним из самых популярных инструментов для начинающих и опытных разработчиков.

### Преимущества Remix

- **Не требует установки** — работает прямо в браузере
- **Встроенный компилятор Solidity** — поддерживает множество версий
- **Мгновенное тестирование** — JavaScript VM для локального запуска
- **Дебаггер транзакций** — пошаговый анализ выполнения
- **Статический анализ** — поиск уязвимостей в коде
- **Интеграция с MetaMask** — деплой в реальные сети

### Доступ к Remix

Remix доступен по адресу: [https://remix.ethereum.org](https://remix.ethereum.org)

Также можно установить desktop-версию или запустить локально:

```bash
# Установка Remix Desktop
npm install -g @remix-project/remix-ide

# Запуск локально
remix-ide
```

---

## Интерфейс Remix IDE

### File Explorer (Файловый менеджер)

Левая панель содержит файловый менеджер для организации проекта:

```
contracts/
├── MyToken.sol
├── Voting.sol
└── interfaces/
    └── IERC20.sol
scripts/
├── deploy.js
└── interact.js
tests/
└── MyToken_test.sol
```

**Возможности File Explorer:**
- Создание/удаление файлов и папок
- Импорт файлов с GitHub и IPFS
- Подключение к локальной файловой системе (Remixd)
- Workspaces — разные проекты в одном браузере

### Editor (Редактор кода)

Центральная область — редактор с подсветкой синтаксиса:

- Подсветка синтаксиса Solidity
- Автодополнение кода
- Подсказки об ошибках в реальном времени
- Поддержка нескольких вкладок
- Быстрый переход к определениям

### Terminal (Терминал)

Нижняя панель показывает:
- Логи компиляции
- Результаты транзакций
- События (events)
- Ошибки и предупреждения

```javascript
// Пример вывода в терминале
[vm] from: 0x5B3...eddC4
to: MyToken.transfer(address,uint256) 0xd91...39138
value: 0 wei
data: 0xa9059...
logs: 1
hash: 0x34e8...
```

### Side Panel (Боковая панель)

Правая часть — панель плагинов:
- Solidity Compiler
- Deploy & Run Transactions
- Debugger
- Solidity Static Analysis
- И многие другие

---

## Компиляция контрактов

### Настройки компилятора

В разделе **Solidity Compiler** настраиваем:

```yaml
Compiler Version: 0.8.20+commit.a1b79de6
Language: Solidity
EVM Version: paris (default)
Optimization: Enable (200 runs)
```

**Основные параметры:**

| Параметр | Описание |
|----------|----------|
| Compiler Version | Версия Solidity (должна совпадать с pragma) |
| EVM Version | Версия виртуальной машины (london, paris, shanghai) |
| Optimization | Оптимизация байткода (уменьшает газ) |
| Runs | Количество предполагаемых вызовов функций |

### Пример контракта для компиляции

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleStorage {
    uint256 private storedValue;

    event ValueChanged(uint256 oldValue, uint256 newValue);

    function set(uint256 newValue) public {
        uint256 oldValue = storedValue;
        storedValue = newValue;
        emit ValueChanged(oldValue, newValue);
    }

    function get() public view returns (uint256) {
        return storedValue;
    }
}
```

### Артефакты компиляции

После компиляции генерируются:
- **ABI** (Application Binary Interface) — описание интерфейса контракта
- **Bytecode** — код для деплоя
- **Runtime Bytecode** — код, хранящийся в блокчейне

```json
// Пример ABI
[
  {
    "inputs": [],
    "name": "get",
    "outputs": [{"type": "uint256"}],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [{"name": "newValue", "type": "uint256"}],
    "name": "set",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

---

## Деплой контрактов

### Окружения для деплоя

В разделе **Deploy & Run Transactions** выбираем окружение:

#### 1. Remix VM (JavaScript VM)

Локальная виртуальная машина для быстрого тестирования:

```
Environment: Remix VM (Cancun)
Accounts: 10 тестовых аккаунтов с 100 ETH
Gas Limit: 3000000
```

**Преимущества:**
- Мгновенный деплой и выполнение
- Не требует газа
- Идеально для отладки

#### 2. Injected Provider (MetaMask)

Подключение через расширение браузера:

```
Environment: Injected Provider - MetaMask
Network: Sepolia Testnet (chainId: 11155111)
Account: 0x742d35Cc6634C0532925a3b844Bc9e7595f...
```

**Использование:**
1. Установите MetaMask
2. Выберите сеть (testnet/mainnet)
3. Remix автоматически подключится

#### 3. Web3 Provider

Прямое подключение к RPC-узлу:

```
Provider URL: http://localhost:8545 (Hardhat/Ganache)
или
Provider URL: https://sepolia.infura.io/v3/YOUR_KEY
```

### Процесс деплоя

```javascript
// 1. Выбираем контракт
Contract: SimpleStorage

// 2. Указываем параметры конструктора (если есть)
Constructor Arguments: [initialValue]

// 3. Нажимаем Deploy

// 4. Результат
status: true Transaction mined
contractAddress: 0xd9145CCE52D386f254917e481eB44e9943F39138
```

### Взаимодействие с контрактом

После деплоя появляется интерфейс взаимодействия:

```
Deployed Contracts:
└── SIMPLESTORAGES AT 0XD91...39138
    ├── [set] uint256 newValue: [____] [transact]
    └── [get] [call] → 0: uint256: 0
```

- **Оранжевые кнопки** — функции, изменяющие состояние (требуют газ)
- **Синие кнопки** — view/pure функции (бесплатные)

---

## Дебаггер транзакций

Remix предоставляет мощный дебаггер для пошагового анализа:

### Запуск дебаггера

1. Выполните транзакцию
2. В терминале нажмите **Debug** рядом с транзакцией
3. Или введите хеш транзакции в плагине Debugger

### Панели дебаггера

```
Instructions:  PUSH1 0x80 PUSH1 0x40 MSTORE...
Solidity Locals:
  newValue: 42
  oldValue: 0
Solidity State:
  storedValue: 42
Stack:
  0x0000...002a
Memory:
  0x00: 0000000000000000000000000000000000000000...
Storage:
  0x00: 0x000000000000000000000000000000000000002a
```

### Шаги отладки

| Кнопка | Действие |
|--------|----------|
| Step Into | Войти в функцию |
| Step Over | Пропустить функцию |
| Step Back | Шаг назад |
| Jump to Exception | Перейти к ошибке |

### Анализ газа

Дебаггер показывает расход газа на каждую операцию:

```
SSTORE (storedValue = newValue)
  Gas used: 20000 (cold storage)
  или
  Gas used: 2900 (warm storage)
```

---

## Плагины Remix

### Solidity Static Analysis

Автоматический поиск уязвимостей и проблем:

```
Security:
✓ Check effects-interaction pattern
✓ Reentrancy vulnerabilities
✓ Selfdestruct detection

Gas & Economy:
✓ Gas costs of functions
✓ Check for costly operations in loops

Miscellaneous:
✓ Constant/View functions
✓ Similar variable names
```

### Unit Testing (Solidity Unit Testing)

Написание тестов на Solidity:

```solidity
// tests/SimpleStorage_test.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "remix_tests.sol";
import "../contracts/SimpleStorage.sol";

contract SimpleStorageTest {
    SimpleStorage storageContract;

    function beforeEach() public {
        storageContract = new SimpleStorage();
    }

    function testInitialValue() public {
        Assert.equal(
            storageContract.get(),
            0,
            "Initial value should be 0"
        );
    }

    function testSetValue() public {
        storageContract.set(42);
        Assert.equal(
            storageContract.get(),
            42,
            "Value should be 42 after set"
        );
    }

    function testMultipleChanges() public {
        storageContract.set(100);
        storageContract.set(200);
        Assert.equal(
            storageContract.get(),
            200,
            "Value should be last set value"
        );
    }
}
```

Запуск тестов через плагин **Solidity Unit Testing**.

### Другие полезные плагины

| Плагин | Назначение |
|--------|------------|
| Etherscan | Верификация контрактов |
| Flattener | Объединение файлов |
| Gas Profiler | Анализ расхода газа |
| Contract Verification | Верификация на Sourcify |
| DGIT | Git-интеграция |

---

## Практические примеры

### Пример 1: Создание и деплой ERC20 токена

```solidity
// contracts/MyToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("MyToken", "MTK") {
        _mint(msg.sender, initialSupply * 10**decimals());
    }
}
```

**Шаги:**
1. Создайте файл `MyToken.sol`
2. Remix автоматически загрузит OpenZeppelin
3. Скомпилируйте с версией 0.8.20
4. Деплой с параметром `1000000` (1 миллион токенов)

### Пример 2: Отладка failed транзакции

```solidity
// contracts/Restricted.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Restricted {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function restrictedAction() public view {
        require(msg.sender == owner, "Not authorized");
        // ... логика
    }
}
```

**Отладка:**
1. Деплой от Account 1
2. Вызов `restrictedAction()` от Account 2
3. Транзакция fail
4. Нажать Debug → увидеть revert на `require`

### Пример 3: Тестирование событий

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EventLogger {
    event Transfer(address indexed from, address indexed to, uint256 value);

    function emitTransfer(address to, uint256 value) public {
        emit Transfer(msg.sender, to, value);
    }
}
```

После вызова `emitTransfer`:
- В терминале видны логи события
- Можно фильтровать по indexed параметрам

---

## Remixd — работа с локальными файлами

Для синхронизации с локальной файловой системой:

```bash
# Установка
npm install -g @remix-project/remixd

# Запуск
remixd -s /path/to/your/project --remix-ide https://remix.ethereum.org
```

В Remix: **File Explorer** → **Connect to Localhost**

---

## Горячие клавиши

| Комбинация | Действие |
|------------|----------|
| Ctrl + S | Компиляция текущего файла |
| Ctrl + Shift + S | Компиляция с выбором версии |
| F7 | Step Into (дебаггер) |
| F8 | Step Over (дебаггер) |
| F10 | Step Back (дебаггер) |

---

## Ограничения Remix

- **Не подходит для больших проектов** — нет полноценной системы зависимостей
- **Ограниченная поддержка Git** — лучше использовать локальные инструменты
- **Браузерные ограничения** — данные могут потеряться при очистке кэша

Для серьезных проектов рекомендуется использовать Remix в сочетании с локальными инструментами (Hardhat, Foundry) или перейти на VS Code.

---

## Полезные ресурсы

- [Официальная документация Remix](https://remix-ide.readthedocs.io/)
- [Remix на GitHub](https://github.com/ethereum/remix-project)
- [Remix Plugin Directory](https://remix-plugins-directory.readthedocs.io/)
