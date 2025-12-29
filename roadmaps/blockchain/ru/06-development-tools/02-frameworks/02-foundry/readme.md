# Foundry

## Введение

**Foundry** - это молниеносно быстрый, портативный и модульный инструментарий для разработки Ethereum приложений, написанный на Rust. Foundry отличается от других фреймворков тем, что тесты пишутся на Solidity, а не на JavaScript/TypeScript, что обеспечивает единую среду разработки и тестирования.

## Зачем нужен Foundry

1. **Скорость** - компиляция и тестирование в разы быстрее JavaScript-based фреймворков
2. **Тесты на Solidity** - тестируйте контракты на том же языке, на котором они написаны
3. **Встроенный fuzzing** - автоматический поиск edge cases
4. **Простота** - минимум конфигурации, максимум функционала
5. **Мощные инструменты** - forge, cast, anvil, chisel
6. **Нативная поддержка форков** - тестирование с реальным состоянием mainnet

## Установка

### Через foundryup (рекомендуется)

```bash
# Установка foundryup
curl -L https://foundry.paradigm.xyz | bash

# Перезагрузка shell или выполнение
source ~/.bashrc  # или ~/.zshrc

# Установка Foundry
foundryup
```

### Обновление

```bash
# Обновление до последней версии
foundryup

# Установка конкретной версии
foundryup --version nightly-2024-01-01
```

### Проверка установки

```bash
forge --version
cast --version
anvil --version
chisel --version
```

## Структура проекта

### Создание нового проекта

```bash
# Создание проекта
forge init my-project
cd my-project

# Создание из шаблона
forge init --template paradigmxyz/forge-template my-project
```

### Структура директорий

```
my-project/
├── src/                # Исходные контракты
│   └── Counter.sol
├── test/               # Тесты (на Solidity)
│   └── Counter.t.sol
├── script/             # Скрипты деплоя
│   └── Counter.s.sol
├── lib/                # Зависимости (git submodules)
│   └── forge-std/
├── out/                # Скомпилированные артефакты
├── cache/              # Кэш компиляции
├── foundry.toml        # Конфигурация
└── remappings.txt      # Маппинг импортов
```

## Конфигурация foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = true

# Настройки тестов
fuzz = { runs = 256 }
invariant = { runs = 256, depth = 15 }

# Форматирование
line_length = 120
tab_width = 4
bracket_spacing = true

# RPC endpoints
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"

# Etherscan API keys
[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }

# Оптимизация для продакшена
[profile.production]
optimizer = true
optimizer_runs = 1000000

# Профиль для CI
[profile.ci]
fuzz = { runs = 10000 }
verbosity = 4
```

## Forge - компиляция и тестирование

### Основные команды компиляции

```bash
# Компиляция проекта
forge build

# Принудительная перекомпиляция
forge build --force

# Компиляция с оптимизацией
forge build --optimize

# Компиляция в режиме production
forge build --profile production

# Проверка размера контрактов
forge build --sizes
```

### Управление зависимостями

```bash
# Установка зависимости
forge install OpenZeppelin/openzeppelin-contracts

# Установка конкретной версии
forge install OpenZeppelin/openzeppelin-contracts@v5.0.0

# Обновление зависимостей
forge update

# Удаление зависимости
forge remove openzeppelin-contracts
```

### Remappings

```bash
# Генерация remappings
forge remappings > remappings.txt
```

```txt
# remappings.txt
@openzeppelin/=lib/openzeppelin-contracts/
forge-std/=lib/forge-std/src/
```

## Тестирование на Solidity

### Базовый тест

```solidity
// test/Counter.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {Counter} from "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;
    address public owner;
    address public user;

    // Вызывается перед каждым тестом
    function setUp() public {
        owner = makeAddr("owner");
        user = makeAddr("user");

        vm.prank(owner);
        counter = new Counter();
    }

    function test_InitialValue() public view {
        assertEq(counter.number(), 0);
    }

    function test_Increment() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    function test_SetNumber() public {
        counter.setNumber(42);
        assertEq(counter.number(), 42);
    }

    // Тест на revert
    function test_RevertWhen_Unauthorized() public {
        vm.prank(user);
        vm.expectRevert("Not owner");
        counter.reset();
    }

    // Тест с событиями
    function test_EmitEvent() public {
        vm.expectEmit(true, false, false, true);
        emit Counter.NumberChanged(42);
        counter.setNumber(42);
    }
}
```

### Запуск тестов

```bash
# Запуск всех тестов
forge test

# Запуск с verbosity (v = больше информации)
forge test -vvvv

# Запуск конкретного теста
forge test --match-test test_Increment

# Запуск тестов в файле
forge test --match-path test/Counter.t.sol

# Запуск тестов контракта
forge test --match-contract CounterTest

# Gas report
forge test --gas-report

# Покрытие кода
forge coverage
```

### Cheatcodes (vm)

Foundry предоставляет мощные cheatcodes для манипуляции состоянием:

```solidity
// test/Cheatcodes.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";

contract CheatcodesTest is Test {

    // Манипуляция адресами
    function test_Prank() public {
        address alice = makeAddr("alice");

        // Следующий вызов будет от alice
        vm.prank(alice);
        // someContract.doSomething();

        // Все последующие вызовы от alice до stopPrank
        vm.startPrank(alice);
        // ... множественные вызовы ...
        vm.stopPrank();
    }

    // Манипуляция балансом
    function test_Deal() public {
        address bob = makeAddr("bob");

        // Установить баланс ETH
        vm.deal(bob, 100 ether);
        assertEq(bob.balance, 100 ether);

        // Для ERC20 токенов
        // deal(address(token), bob, 1000e18);
    }

    // Манипуляция временем
    function test_Time() public {
        // Установить timestamp
        vm.warp(1700000000);
        assertEq(block.timestamp, 1700000000);

        // Перемотать на X секунд
        skip(3600); // +1 час

        // Вернуться назад
        rewind(1800); // -30 минут
    }

    // Манипуляция блоками
    function test_Block() public {
        // Установить номер блока
        vm.roll(1000000);
        assertEq(block.number, 1000000);
    }

    // Ожидание revert
    function test_ExpectRevert() public {
        vm.expectRevert("Error message");
        // revertingFunction();

        // С custom error
        vm.expectRevert(CustomError.selector);
        // revertingFunction();

        // С параметрами
        vm.expectRevert(abi.encodeWithSelector(
            CustomError.selector,
            param1,
            param2
        ));
    }

    // Снимок и восстановление состояния
    function test_Snapshot() public {
        uint256 snapshot = vm.snapshot();

        // ... изменения состояния ...

        // Восстановить состояние
        vm.revertTo(snapshot);
    }

    // Чтение/запись storage напрямую
    function test_Storage() public {
        // Прочитать slot
        bytes32 value = vm.load(address(someContract), bytes32(0));

        // Записать в slot
        vm.store(address(someContract), bytes32(0), bytes32(uint256(42)));
    }
}
```

## Fuzz Testing

Foundry автоматически генерирует входные данные для поиска edge cases:

```solidity
// test/Fuzz.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Token} from "../src/Token.sol";

contract FuzzTest is Test {
    Token public token;

    function setUp() public {
        token = new Token("Test", "TST", 1000000e18);
    }

    // Fuzz test - Foundry автоматически генерирует значения
    function testFuzz_Transfer(address to, uint256 amount) public {
        // Bound ограничивает значения разумным диапазоном
        vm.assume(to != address(0));
        vm.assume(to != address(this));
        amount = bound(amount, 0, token.balanceOf(address(this)));

        uint256 balanceBefore = token.balanceOf(address(this));

        token.transfer(to, amount);

        assertEq(token.balanceOf(address(this)), balanceBefore - amount);
        assertEq(token.balanceOf(to), amount);
    }

    // Fuzz test с несколькими параметрами
    function testFuzz_MintAndBurn(
        address user,
        uint256 mintAmount,
        uint256 burnAmount
    ) public {
        vm.assume(user != address(0));
        mintAmount = bound(mintAmount, 1, 1000000e18);
        burnAmount = bound(burnAmount, 0, mintAmount);

        token.mint(user, mintAmount);

        vm.prank(user);
        token.burn(burnAmount);

        assertEq(token.balanceOf(user), mintAmount - burnAmount);
    }
}
```

### Настройка fuzz тестов

```toml
# foundry.toml
[fuzz]
runs = 256              # Количество итераций
max_test_rejects = 65536
seed = "0x1234"         # Фиксированный seed для воспроизводимости
dictionary_weight = 40  # Вес словаря значений
```

## Invariant Testing

Проверка инвариантов, которые должны всегда выполняться:

```solidity
// test/Invariant.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {StdInvariant} from "forge-std/StdInvariant.sol";
import {Vault} from "../src/Vault.sol";

contract Handler is Test {
    Vault public vault;

    constructor(Vault _vault) {
        vault = _vault;
    }

    function deposit(uint256 amount) public {
        amount = bound(amount, 0, 100 ether);
        vm.deal(address(this), amount);
        vault.deposit{value: amount}();
    }

    function withdraw(uint256 amount) public {
        amount = bound(amount, 0, vault.balanceOf(address(this)));
        vault.withdraw(amount);
    }
}

contract VaultInvariantTest is StdInvariant, Test {
    Vault public vault;
    Handler public handler;

    function setUp() public {
        vault = new Vault();
        handler = new Handler(vault);

        // Указываем, какой контракт тестировать
        targetContract(address(handler));
    }

    // Инвариант: баланс контракта >= сумма всех депозитов
    function invariant_SolvencyCheck() public view {
        assertGe(
            address(vault).balance,
            vault.totalDeposited()
        );
    }

    // Инвариант: totalSupply = sum of balances
    function invariant_TotalSupply() public view {
        // Проверка целостности
    }
}
```

## Cast - взаимодействие с блокчейном

Cast - CLI инструмент для взаимодействия с EVM блокчейнами:

```bash
# Чтение данных
cast call 0x... "balanceOf(address)" 0xYourAddress --rpc-url $RPC_URL

# Отправка транзакции
cast send 0x... "transfer(address,uint256)" 0xTo 1000 \
  --private-key $PRIVATE_KEY \
  --rpc-url $RPC_URL

# Получение баланса
cast balance 0xAddress --rpc-url $RPC_URL

# Конвертация единиц
cast to-wei 1.5 ether      # -> 1500000000000000000
cast from-wei 1000000000000000000 ether  # -> 1

# Кодирование/декодирование
cast abi-encode "transfer(address,uint256)" 0xTo 1000
cast abi-decode "transfer(address,uint256)" 0x...

# Получение кода контракта
cast code 0xContractAddress --rpc-url $RPC_URL

# Чтение storage
cast storage 0xContract 0 --rpc-url $RPC_URL

# Информация о блоке
cast block latest --rpc-url $RPC_URL

# Информация о транзакции
cast tx 0xTxHash --rpc-url $RPC_URL

# Получение ABI с Etherscan
cast etherscan-source 0xContract --etherscan-api-key $KEY

# Вычисление selector
cast sig "transfer(address,uint256)"  # -> 0xa9059cbb

# Keccak256 hash
cast keccak "Hello World"
```

## Anvil - локальная нода

Anvil - это локальная Ethereum нода для разработки:

```bash
# Запуск локальной ноды
anvil

# С форком mainnet
anvil --fork-url $MAINNET_RPC_URL

# С форком на конкретном блоке
anvil --fork-url $MAINNET_RPC_URL --fork-block-number 18500000

# Настройка параметров
anvil \
  --port 8545 \
  --accounts 10 \
  --balance 10000 \
  --block-time 12 \
  --chain-id 31337

# С автоматическим mining
anvil --block-time 0  # Instant mining

# Сохранение состояния
anvil --dump-state state.json
anvil --load-state state.json
```

### Пример использования с fork

```bash
# Терминал 1: Запуск anvil с форком
anvil --fork-url $MAINNET_RPC_URL

# Терминал 2: Взаимодействие
cast call 0x6B175474E89094C44Da98b954EecscdeCB5BE3F3E \
  "balanceOf(address)(uint256)" \
  0xWhale \
  --rpc-url http://127.0.0.1:8545
```

## Chisel - REPL для Solidity

Chisel - интерактивная консоль для Solidity:

```bash
# Запуск REPL
chisel

# В REPL
➜ uint256 x = 100
➜ x * 2
Type: uint256
├ Hex: 0xc8
├ Hex (full word): 0x00000000000000000000000000000000000000000000000000000000000000c8
└ Decimal: 200

➜ address alice = 0x1234...
➜ alice.balance

# Импорт контракта
➜ !source src/Token.sol
➜ Token token = new Token("Test", "TST", 1000)
➜ token.totalSupply()

# Форк mainnet в REPL
chisel --fork-url $MAINNET_RPC_URL
```

## Скрипты деплоя

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console} from "forge-std/Script.sol";
import {Token} from "../src/Token.sol";

contract DeployScript is Script {
    function setUp() public {}

    function run() public {
        // Загрузка приватного ключа из env
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);

        console.log("Deploying from:", deployer);
        console.log("Balance:", deployer.balance);

        vm.startBroadcast(deployerPrivateKey);

        Token token = new Token("MyToken", "MTK", 1000000e18);
        console.log("Token deployed at:", address(token));

        vm.stopBroadcast();
    }
}
```

### Запуск скриптов

```bash
# Симуляция (без отправки транзакций)
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# Реальный деплой
forge script script/Deploy.s.sol \
  --rpc-url $RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY

# С подтверждением
forge script script/Deploy.s.sol \
  --rpc-url $RPC_URL \
  --broadcast \
  --verify \
  -vvvv
```

## Верификация контрактов

```bash
# Верификация после деплоя
forge verify-contract \
  --chain-id 1 \
  --num-of-optimizations 200 \
  --constructor-args $(cast abi-encode "constructor(string,string,uint256)" "MyToken" "MTK" 1000000000000000000000000) \
  0xContractAddress \
  src/Token.sol:Token \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

## Лучшие практики

1. **Используйте forge-std** - стандартная библиотека с полезными утилитами
2. **Пишите fuzz тесты** - находите edge cases автоматически
3. **Используйте invariant тесты** - проверяйте критические свойства
4. **Профилируйте gas** - `forge test --gas-report`
5. **Используйте snapshots** - для сложных тестовых сценариев
6. **Организуйте handlers** - для invariant тестирования

## Сравнение с Hardhat

| Аспект | Foundry | Hardhat |
|--------|---------|---------|
| Язык тестов | Solidity | JavaScript/TypeScript |
| Скорость | Очень быстро | Медленнее |
| Fuzzing | Встроенный | Требует плагины |
| Экосистема | Растущая | Большая |
| Обучение | Проще для Solidity разработчиков | Проще для JS разработчиков |

## Ресурсы

- [Foundry Book](https://book.getfoundry.sh/)
- [Foundry GitHub](https://github.com/foundry-rs/foundry)
- [forge-std](https://github.com/foundry-rs/forge-std)
- [Awesome Foundry](https://github.com/crisgarner/awesome-foundry)
