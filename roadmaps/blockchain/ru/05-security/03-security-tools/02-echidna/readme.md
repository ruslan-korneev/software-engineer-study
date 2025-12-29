# Echidna

Echidna - это property-based фаззер для смарт-контрактов от Trail of Bits. Он автоматически генерирует транзакции для поиска нарушений заданных свойств (инвариантов).

## Что такое Property-based Testing

Property-based тестирование отличается от unit-тестирования:

| Unit Testing | Property Testing |
|--------------|------------------|
| Тестирует конкретные входные данные | Тестирует свойства с любыми входами |
| `transfer(100)` должен работать | "Баланс никогда не становится отрицательным" |
| Проверяет ожидаемый результат | Проверяет инвариант |
| Ограничен воображением разработчика | Находит edge cases автоматически |

## Установка

### Бинарные релизы (рекомендуется)

```bash
# macOS
brew install echidna

# Linux (скачать бинарник)
wget https://github.com/crytic/echidna/releases/latest/download/echidna-x86_64-linux.tar.gz
tar -xzf echidna-x86_64-linux.tar.gz
sudo mv echidna /usr/local/bin/

# Проверка установки
echidna --version
```

### Через Docker

```bash
docker pull trailofbits/eth-security-toolbox
docker run -it -v $(pwd):/src trailofbits/eth-security-toolbox
# Внутри контейнера echidna уже установлен
```

### Сборка из исходников

```bash
# Требуется установленный stack (Haskell)
git clone https://github.com/crytic/echidna
cd echidna
stack install
```

### Зависимости

Echidna требует solc (компилятор Solidity):

```bash
# Установка через solc-select
pip install solc-select
solc-select install 0.8.20
solc-select use 0.8.20
```

## Основы работы

### Структура теста

Echidna ищет функции, начинающиеся с `echidna_`, и проверяет, что они всегда возвращают `true`:

```solidity
// TestContract.sol
pragma solidity ^0.8.20;

contract Counter {
    uint256 public count;

    function increment() public {
        count += 1;
    }

    function decrement() public {
        require(count > 0, "Cannot decrement below 0");
        count -= 1;
    }
}

contract TestCounter is Counter {
    // Свойство: count никогда не становится отрицательным
    // (для uint256 это overflow)
    function echidna_count_non_negative() public view returns (bool) {
        return count >= 0;  // Всегда true для uint256
    }

    // Свойство: count ограничен сверху
    function echidna_count_bounded() public view returns (bool) {
        return count < 1000;  // Echidna найдет нарушение
    }
}
```

### Запуск

```bash
# Базовый запуск
echidna TestContract.sol --contract TestCounter

# С указанием конфигурации
echidna TestContract.sol --contract TestCounter --config echidna.yaml

# С использованием Foundry
echidna . --contract TestCounter
```

### Пример вывода

```
Analyzing contract: TestCounter
echidna_count_non_negative: passing
echidna_count_bounded: failed!💥
  Call sequence:
    increment()
    increment()
    increment()
    ... (1000 calls)

Unique instructions: 45
Unique codehashes: 1
Corpus size: 3
Seed: 1234567890
```

## Конфигурация

### Файл конфигурации (echidna.yaml)

```yaml
# Режим тестирования
testMode: "property"  # property, assertion, optimization, overflow

# Лимиты
testLimit: 50000       # Максимальное количество транзакций
seqLen: 100            # Максимальная длина последовательности
shrinkLimit: 5000      # Лимит на shrinking

# Настройки контракта
deployer: "0x10000"                    # Адрес деплоера
sender: ["0x10001", "0x10002"]         # Возможные отправители
contractAddr: "0x00a329c0648769a73"    # Адрес контракта

# Значения
balanceContract: 0                     # Начальный баланс контракта
balanceAddr: 0xffffffff               # Баланс каждого sender
maxValue: 100000000000000000          # Максимальное value в транзакции

# Покрытие
coverage: true
corpusDir: "corpus"

# Таймауты
timeout: 300           # Общий таймаут в секундах
maxTimeDelay: 604800   # Макс. прыжок времени (7 дней)
maxBlockDelay: 60480   # Макс. прыжок блоков

# Фильтрация функций
filterBlacklist: true
filterFunctions: ["excludedFunction"]

# Вывод
format: "text"         # text, json, none
quiet: false
```

### Режимы тестирования

```yaml
# 1. Property mode (по умолчанию)
# Ищет нарушение echidna_* функций
testMode: "property"

# 2. Assertion mode
# Ищет нарушение assert() в контракте
testMode: "assertion"

# 3. Optimization mode
# Максимизирует значение echidna_* функций
testMode: "optimization"

# 4. Overflow mode
# Ищет integer overflow/underflow
testMode: "overflow"
```

## Написание свойств

### Инварианты (Invariants)

Инварианты - это условия, которые всегда должны быть истинными:

```solidity
contract Token {
    mapping(address => uint256) public balances;
    uint256 public totalSupply;

    function mint(address to, uint256 amount) external {
        balances[to] += amount;
        totalSupply += amount;
    }

    function burn(address from, uint256 amount) external {
        require(balances[from] >= amount);
        balances[from] -= amount;
        totalSupply -= amount;
    }

    function transfer(address from, address to, uint256 amount) external {
        require(balances[from] >= amount);
        balances[from] -= amount;
        balances[to] += amount;
    }
}

contract TokenTest is Token {
    // Инвариант: сумма балансов == totalSupply
    // (проверить это свойство сложно, нужен массив адресов)

    address[] internal holders;

    function addHolder(address h) internal {
        for (uint i = 0; i < holders.length; i++) {
            if (holders[i] == h) return;
        }
        holders.push(h);
    }

    function mint(address to, uint256 amount) external override {
        addHolder(to);
        super.mint(to, amount);
    }

    function echidna_total_supply_invariant() public view returns (bool) {
        uint256 sum = 0;
        for (uint i = 0; i < holders.length; i++) {
            sum += balances[holders[i]];
        }
        return sum == totalSupply;
    }
}
```

### Функциональные свойства

```solidity
contract Vault {
    mapping(address => uint256) public deposits;

    function deposit() external payable {
        deposits[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) external {
        require(deposits[msg.sender] >= amount);
        deposits[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}

contract VaultTest is Vault {
    // Свойство: пользователь не может вывести больше, чем внес
    function echidna_no_free_money() public view returns (bool) {
        // Баланс контракта >= суммы депозитов
        return address(this).balance >= deposits[msg.sender];
    }

    // Свойство: баланс контракта всегда положительный
    function echidna_solvent() public view returns (bool) {
        return address(this).balance >= 0;
    }
}
```

### Свойства безопасности

```solidity
contract AccessControl {
    address public owner;
    mapping(address => bool) public admins;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function addAdmin(address admin) external onlyOwner {
        admins[admin] = true;
    }

    function removeAdmin(address admin) external onlyOwner {
        admins[admin] = false;
    }

    function transferOwnership(address newOwner) external onlyOwner {
        owner = newOwner;
    }
}

contract AccessControlTest is AccessControl {
    address internal constant ATTACKER = address(0xdeadbeef);

    constructor() {
        // Attacker не owner и не admin
        require(owner != ATTACKER);
        require(!admins[ATTACKER]);
    }

    // Свойство: атакующий не может стать owner
    function echidna_attacker_not_owner() public view returns (bool) {
        return owner != ATTACKER;
    }

    // Свойство: атакующий не может стать admin
    function echidna_attacker_not_admin() public view returns (bool) {
        return !admins[ATTACKER];
    }
}
```

## Assertion Mode

Вместо `echidna_*` функций можно использовать `assert()`:

```solidity
contract MathLib {
    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        uint256 c = a + b;
        assert(c >= a);  // Echidna проверит это
        return c;
    }

    function safeSub(uint256 a, uint256 b) public pure returns (uint256) {
        assert(b <= a);  // Echidna проверит это
        return a - b;
    }

    function safeDiv(uint256 a, uint256 b) public pure returns (uint256) {
        assert(b > 0);   // Echidna проверит это
        return a / b;
    }
}
```

Запуск:

```bash
echidna MathLib.sol --contract MathLib --test-mode assertion
```

## Optimization Mode

Режим оптимизации ищет максимальное значение функции:

```solidity
contract GasOptimization {
    uint256 public gasUsed;

    function complexOperation(uint256[] calldata data) external {
        uint256 startGas = gasleft();

        // Сложные вычисления
        uint256 sum = 0;
        for (uint i = 0; i < data.length; i++) {
            sum += data[i];
        }

        gasUsed = startGas - gasleft();
    }

    // Echidna будет максимизировать это значение
    function echidna_max_gas() public view returns (int256) {
        return int256(gasUsed);
    }
}
```

Запуск:

```bash
echidna GasOptimization.sol --contract GasOptimization --test-mode optimization
```

## Продвинутые техники

### Stateful Testing

```solidity
contract StateMachine {
    enum State { Idle, Active, Paused, Finished }
    State public state = State.Idle;

    function start() external {
        require(state == State.Idle);
        state = State.Active;
    }

    function pause() external {
        require(state == State.Active);
        state = State.Paused;
    }

    function resume() external {
        require(state == State.Paused);
        state = State.Active;
    }

    function finish() external {
        require(state == State.Active);
        state = State.Finished;
    }
}

contract StateMachineTest is StateMachine {
    // Свойство: нельзя перейти из Finished
    function echidna_finished_is_final() public view returns (bool) {
        // Если мы в Finished, то остаемся там
        return state != State.Finished || true;
    }

    // Свойство: нельзя перейти из Idle в Finished напрямую
    bool internal wasActive = false;

    function start() external override {
        super.start();
        wasActive = true;
    }

    function echidna_must_be_active_before_finish() public view returns (bool) {
        return state != State.Finished || wasActive;
    }
}
```

### Multi-contract Testing

```solidity
// Token.sol
contract Token {
    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }
}

// Vault.sol
contract Vault {
    Token public token;
    mapping(address => uint256) public deposits;

    constructor(Token _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        token.transferFrom(msg.sender, address(this), amount);
        deposits[msg.sender] += amount;
    }
}

// Test.sol
contract MultiTest {
    Token token;
    Vault vault;

    constructor() {
        token = new Token();
        vault = new Vault(token);
    }

    function echidna_vault_balance_matches() public view returns (bool) {
        return token.balanceOf(address(vault)) >= vault.deposits(address(this));
    }
}
```

### Corpus и Seed

```yaml
# echidna.yaml
corpusDir: "corpus"      # Директория для сохранения найденных интересных входов
coverage: true           # Включить coverage-guided fuzzing
seed: 12345             # Фиксированный seed для воспроизводимости
```

Структура corpus:

```
corpus/
├── coverage/           # Входы для покрытия кода
│   ├── 0x1234...
│   └── 0x5678...
└── reproducers/        # Входы, нарушающие свойства
    ├── echidna_test_1/
    │   └── 0xabcd...
    └── echidna_test_2/
        └── 0xefgh...
```

### Integration с Foundry

```solidity
// test/EchidnaTest.t.sol
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {MyContract} from "../src/MyContract.sol";

contract EchidnaTest is MyContract {
    function echidna_invariant() public view returns (bool) {
        // Ваше свойство
        return true;
    }
}
```

Запуск:

```bash
echidna . --contract EchidnaTest
```

## Интеграция в CI/CD

### GitHub Actions

```yaml
# .github/workflows/echidna.yml
name: Echidna Fuzzing

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # Ежедневный фаззинг

jobs:
  echidna:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Build contracts
        run: forge build

      - name: Install Echidna
        run: |
          wget https://github.com/crytic/echidna/releases/latest/download/echidna-x86_64-linux.tar.gz
          tar -xzf echidna-x86_64-linux.tar.gz
          sudo mv echidna /usr/local/bin/

      - name: Run Echidna
        run: echidna . --contract MyTest --config echidna.yaml

      - name: Upload corpus
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: echidna-corpus
          path: corpus/
```

### Конфигурация для CI

```yaml
# echidna-ci.yaml
testLimit: 10000         # Меньше для CI
timeout: 300             # 5 минут максимум
coverage: true
corpusDir: "corpus"
format: "text"
```

## Примеры реальных уязвимостей

### Пример 1: Integer Overflow

```solidity
contract Vulnerable {
    mapping(address => uint256) public balances;

    // VULNERABLE: overflow при большом amount
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] - amount >= 0);  // Всегда true для uint256!
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}

contract VulnerableTest is Vulnerable {
    constructor() {
        balances[msg.sender] = 100;
    }

    // Echidna найдет: withdraw с amount > 100 приведет к underflow
    function echidna_no_underflow() public view returns (bool) {
        return balances[msg.sender] <= 100;  // Начальное значение
    }
}
```

### Пример 2: Access Control Bypass

```solidity
contract Vulnerable {
    address public owner;
    bool public initialized;

    function initialize(address _owner) external {
        require(!initialized);
        owner = _owner;
        initialized = true;
    }

    function privilegedAction() external {
        require(msg.sender == owner);
        // Критическое действие
    }
}

contract VulnerableTest is Vulnerable {
    address constant ATTACKER = address(0xbad);

    function echidna_attacker_not_owner() public view returns (bool) {
        return owner != ATTACKER;
    }
}
```

## Best Practices

### 1. Начинайте с простых свойств

```solidity
// Простые свойства, которые легко проверить
function echidna_total_supply_not_zero() public view returns (bool) {
    return totalSupply > 0 || !initialized;
}
```

### 2. Используйте константы для специальных адресов

```solidity
address constant ADMIN = address(0x1);
address constant USER = address(0x2);
address constant ATTACKER = address(0xbad);
```

### 3. Тестируйте граничные условия

```solidity
function echidna_balance_bounded() public view returns (bool) {
    return balances[msg.sender] <= MAX_SUPPLY;
}
```

### 4. Комбинируйте с unit-тестами

Echidna находит то, что вы не подумали тестировать. Unit-тесты проверяют известные случаи.

### 5. Используйте coverage для оценки

```bash
echidna . --contract Test --coverage
```

## Ограничения Echidna

1. **Не проверяет внешние вызовы** - только код тестируемого контракта
2. **Медленнее unit-тестов** - фаззинг требует времени
3. **Требует написания свойств** - нужно думать об инвариантах
4. **Ложные срабатывания** - иногда находит "проблемы" в тестовом окружении

## Ресурсы

- [Официальная документация](https://github.com/crytic/echidna)
- [Building Secure Contracts - Testing Guide](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna)
- [Echidna Exercises](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna/exercises)
- [Trail of Bits Blog](https://blog.trailofbits.com/)
- [Fuzzing Smart Contracts (видео)](https://www.youtube.com/watch?v=QofNQxW_K08)
