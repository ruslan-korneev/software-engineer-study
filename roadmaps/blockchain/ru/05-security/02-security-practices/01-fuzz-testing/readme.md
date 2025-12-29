# Fuzz-тестирование (Фаззинг)

Fuzz-тестирование — это методика автоматизированного тестирования, при которой программа подвергается воздействию большого количества случайных или полуслучайных входных данных с целью обнаружения ошибок, уязвимостей и неожиданного поведения.

## Концепция фаззинга в смарт-контрактах

В контексте блокчейн-разработки фаззинг позволяет:

1. **Проверить инварианты** — свойства, которые должны выполняться всегда
2. **Найти edge cases** — граничные условия, которые сложно предусмотреть вручную
3. **Обнаружить state-dependent баги** — ошибки, возникающие в определённых состояниях контракта
4. **Проверить арифметические операции** — overflow, underflow, деление на ноль

```
┌────────────────────────────────────────────────────────────────┐
│                    Принцип работы фаззера                      │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌─────────────┐      ┌────────────┐      ┌───────────────┐  │
│   │  Генератор  │─────▶│   Смарт-   │─────▶│   Проверка    │  │
│   │   входов    │      │  контракт  │      │  инвариантов  │  │
│   └─────────────┘      └────────────┘      └───────────────┘  │
│          │                   │                    │           │
│          │                   │                    │           │
│          ▼                   ▼                    ▼           │
│   ┌─────────────┐      ┌────────────┐      ┌───────────────┐  │
│   │  Мутация    │◀─────│  Покрытие  │◀─────│   Нарушено?   │  │
│   │   входов    │      │   кода     │      │   Да → Баг!   │  │
│   └─────────────┘      └────────────┘      └───────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Property-based тестирование

В отличие от unit-тестов, где проверяется конкретный сценарий, property-based тестирование проверяет **свойства** системы на множестве входных данных.

### Пример: Unit тест vs Property тест

```solidity
// UNIT ТЕСТ: проверяем конкретные значения
function test_deposit() public {
    vault.deposit(100);
    assertEq(vault.balanceOf(address(this)), 100);
}

// PROPERTY ТЕСТ: проверяем свойство для любого значения
function testFuzz_deposit(uint256 amount) public {
    uint256 balanceBefore = vault.balanceOf(address(this));
    vault.deposit(amount);
    uint256 balanceAfter = vault.balanceOf(address(this));

    // Свойство: баланс должен увеличиться на сумму депозита
    assertEq(balanceAfter, balanceBefore + amount);
}
```

## Инварианты

**Инвариант** — это условие, которое должно выполняться всегда, независимо от последовательности операций.

### Типы инвариантов

```
┌─────────────────────────────────────────────────────────────────┐
│                     Типы инвариантов                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ГЛОБАЛЬНЫЕ ИНВАРИАНТЫ                                       │
│     • Общий supply токена = сумма всех балансов                 │
│     • TVL контракта ≥ 0                                         │
│     • Owner контракта ≠ address(0)                              │
│                                                                 │
│  2. ИНВАРИАНТЫ СОСТОЯНИЯ                                        │
│     • После pause() все transfers заблокированы                 │
│     • Баланс пользователя ≤ totalSupply                         │
│     • Активные stakes ≤ depositedTokens                         │
│                                                                 │
│  3. ПЕРЕХОДНЫЕ ИНВАРИАНТЫ                                       │
│     • deposit() увеличивает баланс                              │
│     • withdraw() уменьшает баланс                               │
│     • transfer() не меняет totalSupply                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Примеры инвариантов для ERC20

```solidity
// Инвариант 1: Сумма всех балансов == totalSupply
function invariant_totalSupplyEqualsBalances() public {
    uint256 sumOfBalances = 0;
    for (uint256 i = 0; i < actors.length; i++) {
        sumOfBalances += token.balanceOf(actors[i]);
    }
    assertEq(token.totalSupply(), sumOfBalances);
}

// Инвариант 2: Баланс любого адреса <= totalSupply
function invariant_balanceNeverExceedsTotalSupply() public {
    for (uint256 i = 0; i < actors.length; i++) {
        assertLe(token.balanceOf(actors[i]), token.totalSupply());
    }
}

// Инвариант 3: totalSupply не может быть отрицательным (в Solidity это автоматически)
function invariant_totalSupplyNonNegative() public {
    assertGe(token.totalSupply(), 0);
}
```

## Echidna

**Echidna** — это fuzzer для смарт-контрактов на Solidity от Trail of Bits. Использует property-based подход и грамматическое фаззирование.

### Установка

```bash
# Через pip
pip install crytic-compile slither-analyzer
pip install echidna

# Через Docker
docker pull trailofbits/echidna

# Через Homebrew (macOS)
brew install echidna
```

### Структура теста Echidna

```solidity
// contracts/EchidnaTest.sol
pragma solidity ^0.8.0;

import "./MyToken.sol";

contract EchidnaTest {
    MyToken token;

    // Echidna будет вызывать функции начинающиеся с "echidna_"
    // и ожидает что они вернут true

    constructor() {
        token = new MyToken(1000000);
    }

    // ИНВАРИАНТ: totalSupply никогда не должен быть нулём после создания
    function echidna_totalSupply_positive() public view returns (bool) {
        return token.totalSupply() > 0;
    }

    // ИНВАРИАНТ: баланс создателя <= totalSupply
    function echidna_balance_lte_supply() public view returns (bool) {
        return token.balanceOf(address(this)) <= token.totalSupply();
    }

    // Функция для фаззинга (Echidna будет её вызывать с разными параметрами)
    function test_transfer(address to, uint256 amount) public {
        // Это не инвариант, это действие которое фаззер будет выполнять
        if (token.balanceOf(address(this)) >= amount) {
            token.transfer(to, amount);
        }
    }
}
```

### Конфигурация Echidna

```yaml
# echidna.yaml
testMode: "assertion"         # assertion | property | optimization
testLimit: 50000              # Количество тестовых транзакций
seqLen: 100                   # Длина последовательности вызовов
shrinkLimit: 5000             # Лимит на упрощение контрпримера
coverage: true                # Включить отчёт о покрытии
corpusDir: "corpus"           # Директория для corpus
deployer: "0x10000"           # Адрес деплоера
sender:                       # Адреса отправителей
  - "0x20000"
  - "0x30000"
filterBlacklist: true         # Использовать чёрный список функций
filterFunctions:              # Функции для исключения
  - "excludedFunction"
```

### Запуск Echidna

```bash
# Базовый запуск
echidna contracts/EchidnaTest.sol --contract EchidnaTest

# С конфигурацией
echidna contracts/EchidnaTest.sol --contract EchidnaTest --config echidna.yaml

# С отчётом о покрытии
echidna contracts/EchidnaTest.sol --contract EchidnaTest --coverage

# Режим assertion (проверка require/assert)
echidna contracts/EchidnaTest.sol --contract EchidnaTest --test-mode assertion
```

### Пример: Поиск уязвимости с Echidna

```solidity
// Уязвимый контракт
contract VulnerableVault {
    mapping(address => uint256) public balances;
    uint256 public totalDeposited;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
        totalDeposited += msg.value;
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        // БАГ: забыли уменьшить totalDeposited!
        payable(msg.sender).transfer(amount);
    }
}

// Тест Echidna
contract VaultTest is VulnerableVault {
    constructor() payable {
        // Начальный депозит для тестирования
    }

    // Этот инвариант обнаружит баг!
    function echidna_solvency() public view returns (bool) {
        return address(this).balance >= totalDeposited;
    }

    function test_deposit() public payable {
        deposit();
    }

    function test_withdraw(uint256 amount) public {
        if (balances[msg.sender] >= amount) {
            withdraw(amount);
        }
    }
}
```

## Foundry Fuzz Testing

**Foundry** (forge) имеет встроенную поддержку фаззинга, которая проще в использовании, чем Echidna.

### Fuzz-тесты в Foundry

```solidity
// test/Token.t.sol
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/Token.sol";

contract TokenFuzzTest is Test {
    Token token;
    address alice = address(0x1);
    address bob = address(0x2);

    function setUp() public {
        token = new Token(1000000);
        token.transfer(alice, 100000);
    }

    // Базовый fuzz-тест: параметр 'amount' будет фаззироваться
    function testFuzz_transfer(uint256 amount) public {
        // Ограничиваем входные данные
        vm.assume(amount > 0);
        vm.assume(amount <= token.balanceOf(alice));

        uint256 aliceBefore = token.balanceOf(alice);
        uint256 bobBefore = token.balanceOf(bob);

        vm.prank(alice);
        token.transfer(bob, amount);

        // Проверяем свойства
        assertEq(token.balanceOf(alice), aliceBefore - amount);
        assertEq(token.balanceOf(bob), bobBefore + amount);
    }

    // Fuzz с несколькими параметрами
    function testFuzz_transferFromMultiple(
        address from,
        address to,
        uint256 amount
    ) public {
        // Исключаем невалидные адреса
        vm.assume(from != address(0));
        vm.assume(to != address(0));
        vm.assume(from != to);
        vm.assume(amount > 0);

        // Даём токены отправителю
        deal(address(token), from, amount);

        uint256 totalBefore = token.totalSupply();

        vm.prank(from);
        token.transfer(to, amount);

        // Инвариант: totalSupply не меняется
        assertEq(token.totalSupply(), totalBefore);
    }
}
```

### Bound вместо Assume

```solidity
// Плохо: много пропущенных тестов из-за assume
function testFuzz_bad(uint256 amount) public {
    vm.assume(amount > 0);
    vm.assume(amount < 1000);  // Большинство случайных чисел не попадёт
    // ...
}

// Хорошо: bound ограничивает диапазон
function testFuzz_good(uint256 amount) public {
    amount = bound(amount, 1, 999);  // Все значения используются
    // ...
}
```

### Invariant тесты в Foundry

```solidity
// test/invariants/TokenInvariant.t.sol
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/Token.sol";

// Handler — контракт, определяющий какие действия может совершать фаззер
contract TokenHandler is Test {
    Token public token;
    address[] public actors;

    // Ghost variables для отслеживания состояния
    uint256 public ghost_mintedSum;
    uint256 public ghost_burnedSum;

    constructor(Token _token) {
        token = _token;
        actors.push(address(0x1));
        actors.push(address(0x2));
        actors.push(address(0x3));
    }

    function mint(uint256 actorSeed, uint256 amount) external {
        amount = bound(amount, 0, 1e24);
        address actor = actors[actorSeed % actors.length];

        token.mint(actor, amount);
        ghost_mintedSum += amount;
    }

    function burn(uint256 actorSeed, uint256 amount) external {
        address actor = actors[actorSeed % actors.length];
        amount = bound(amount, 0, token.balanceOf(actor));

        if (amount == 0) return;

        vm.prank(actor);
        token.burn(amount);
        ghost_burnedSum += amount;
    }

    function transfer(
        uint256 fromSeed,
        uint256 toSeed,
        uint256 amount
    ) external {
        address from = actors[fromSeed % actors.length];
        address to = actors[toSeed % actors.length];
        amount = bound(amount, 0, token.balanceOf(from));

        if (amount == 0 || from == to) return;

        vm.prank(from);
        token.transfer(to, amount);
    }
}

contract TokenInvariantTest is Test {
    Token token;
    TokenHandler handler;

    function setUp() public {
        token = new Token(0);
        handler = new TokenHandler(token);

        // Указываем Foundry какой контракт фаззить
        targetContract(address(handler));
    }

    // Инвариант: totalSupply = minted - burned
    function invariant_supplyEquation() public {
        assertEq(
            token.totalSupply(),
            handler.ghost_mintedSum() - handler.ghost_burnedSum()
        );
    }

    // Инвариант: сумма балансов = totalSupply
    function invariant_sumOfBalances() public {
        uint256 sum = 0;
        address[] memory actors = new address[](3);
        actors[0] = address(0x1);
        actors[1] = address(0x2);
        actors[2] = address(0x3);

        for (uint256 i = 0; i < actors.length; i++) {
            sum += token.balanceOf(actors[i]);
        }

        assertEq(sum, token.totalSupply());
    }
}
```

### Конфигурация Foundry для фаззинга

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[fuzz]
runs = 256                      # Количество запусков fuzz-теста
max_test_rejects = 65536        # Максимум отклонённых входов (assume)
seed = "0x1234"                 # Seed для воспроизводимости
dictionary_weight = 40          # Вес словаря в генерации
include_storage = true          # Включать storage в словарь
include_push_bytes = true       # Включать push bytes в словарь

[invariant]
runs = 256                      # Количество запусков invariant теста
depth = 15                      # Глубина последовательности вызовов
fail_on_revert = false          # Считать ли revert провалом
call_override = false           # Переопределять ли вызовы
dictionary_weight = 80          # Вес словаря
include_storage = true
include_push_bytes = true
shrink_run_limit = 5000         # Лимит упрощения
```

### Запуск тестов

```bash
# Запустить все fuzz-тесты
forge test --match-test "testFuzz"

# Запустить с большим количеством итераций
forge test --match-test "testFuzz" --fuzz-runs 10000

# Запустить invariant тесты
forge test --match-test "invariant"

# С подробным выводом
forge test --match-test "testFuzz" -vvvv

# Посмотреть покрытие
forge coverage --match-test "testFuzz"
```

## Продвинутые техники

### Stateful Fuzzing

Сохранение состояния между вызовами для более глубокого исследования:

```solidity
contract StatefulFuzzTest is Test {
    MyProtocol protocol;

    // Массив для хранения истории действий
    struct Action {
        uint8 actionType;
        uint256 amount;
        address user;
    }
    Action[] public history;

    function testFuzz_stateful(
        uint8[] calldata actionTypes,
        uint256[] calldata amounts
    ) public {
        for (uint i = 0; i < actionTypes.length && i < 100; i++) {
            uint8 action = actionTypes[i] % 4;
            uint256 amount = bound(amounts[i], 0, 1e20);

            if (action == 0) {
                // deposit
            } else if (action == 1) {
                // withdraw
            } else if (action == 2) {
                // stake
            } else {
                // unstake
            }

            // Проверяем инварианты после каждого действия
            _checkInvariants();
        }
    }

    function _checkInvariants() internal {
        // Инварианты
    }
}
```

### Guided Fuzzing с Seed

```solidity
// Использование seed для воспроизводимости
function testFuzz_reproducible(uint256 seed) public {
    // При фиксированном seed результаты будут одинаковыми
    uint256 deterministicValue = uint256(keccak256(abi.encode(seed, "salt")));
    // ...
}
```

## Сравнение Echidna и Foundry

| Аспект | Echidna | Foundry |
|--------|---------|---------|
| Язык | Haskell | Rust |
| Скорость | Медленнее | Быстрее |
| Настройка | Сложнее | Проще |
| Грамматики | Да | Нет |
| Shrinking | Мощный | Базовый |
| Интеграция | Отдельный инструмент | Часть фреймворка |
| Corpus | Сохраняется | Нет |
| Coverage | Подробный | Базовый |

## Best Practices

### 1. Начните с простых инвариантов

```solidity
// Простой инвариант для начала
function invariant_alwaysTrue() public pure returns (bool) {
    return true;  // Если это падает — проблема в setup
}
```

### 2. Используйте Ghost Variables

```solidity
// Ghost variables отслеживают состояние вне контракта
uint256 public ghost_totalDeposited;

function deposit(uint256 amount) external {
    // ... реальная логика ...
    ghost_totalDeposited += amount;
}

function invariant_ghostMatchesReal() public {
    assertEq(protocol.totalDeposited(), ghost_totalDeposited);
}
```

### 3. Ограничивайте входные данные разумно

```solidity
// Используйте bound вместо assume где возможно
function testFuzz_bounded(uint256 amount) public {
    amount = bound(amount, 1, type(uint128).max);
    // ...
}
```

### 4. Проверяйте покрытие кода

```bash
# Foundry
forge coverage --match-test "testFuzz"

# Echidna с покрытием
echidna . --contract Test --coverage
```

### 5. Сохраняйте failing cases

```bash
# Echidna автоматически сохраняет в corpus
echidna . --contract Test --corpus-dir corpus

# Foundry — записать seed вручную при провале
# FOUNDRY_FUZZ_SEED=0x... forge test
```

## Ресурсы

- [Echidna Documentation](https://github.com/crytic/echidna)
- [Building Secure Contracts — Fuzzing](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna)
- [Foundry Book — Fuzz Testing](https://book.getfoundry.sh/forge/fuzz-testing)
- [Foundry Book — Invariant Testing](https://book.getfoundry.sh/forge/invariant-testing)
- [Trail of Bits Blog](https://blog.trailofbits.com/)
