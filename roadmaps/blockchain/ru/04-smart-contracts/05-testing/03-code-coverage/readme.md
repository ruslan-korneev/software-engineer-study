# Покрытие кода (Code Coverage)

## Введение

Code Coverage (покрытие кода) — метрика, показывающая какой процент кода выполняется во время тестирования. Для смарт-контрактов высокое покрытие критически важно, так как непротестированный код может содержать уязвимости, приводящие к потере средств.

## Метрики покрытия

### Основные типы покрытия

```
┌────────────────────────────────────────────────────────────────────┐
│                     Типы Code Coverage                             │
├──────────────────┬─────────────────────────────────────────────────┤
│ Line Coverage    │ % строк кода, которые были выполнены            │
├──────────────────┼─────────────────────────────────────────────────┤
│ Branch Coverage  │ % условных переходов (if/else), которые         │
│                  │ были проверены в обе стороны                    │
├──────────────────┼─────────────────────────────────────────────────┤
│ Function Coverage│ % функций, которые были вызваны хотя бы раз     │
├──────────────────┼─────────────────────────────────────────────────┤
│ Statement Cover. │ % инструкций (statements), которые выполнились  │
└──────────────────┴─────────────────────────────────────────────────┘
```

### Пример различия метрик

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Example {
    uint256 public value;

    function setValue(uint256 _value, bool _double) external {
        if (_double) {           // Branch 1: true/false
            value = _value * 2;  // Line 1
        } else {
            value = _value;      // Line 2
        }

        require(value > 0, "Value must be positive"); // Line 3
    }
}
```

**Тест 1: только `setValue(5, true)`**
- Line Coverage: 66% (Line 1 и 3 выполнены, Line 2 — нет)
- Branch Coverage: 50% (только true ветка)
- Function Coverage: 100% (функция вызвана)

**Тест 2: `setValue(5, true)` + `setValue(5, false)`**
- Line Coverage: 100%
- Branch Coverage: 100%
- Function Coverage: 100%

## Инструменты покрытия

### Hardhat Coverage (solidity-coverage)

```bash
# Установка
npm install --save-dev solidity-coverage

# hardhat.config.js уже включает coverage через hardhat-toolbox
```

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20",
  // Настройка coverage
  networks: {
    hardhat: {
      // Coverage использует свой инструментированный EVM
    }
  }
};
```

**Запуск:**

```bash
npx hardhat coverage

# С конкретными файлами
npx hardhat coverage --testfiles "test/Token.test.js"
```

**Пример вывода:**

```
----------------------|----------|----------|----------|----------|
File                  |  % Stmts | % Branch |  % Funcs |  % Lines |
----------------------|----------|----------|----------|----------|
 contracts/           |    94.12 |    85.71 |      100 |    94.44 |
  Token.sol           |      100 |      100 |      100 |      100 |
  Vault.sol           |    88.89 |       75 |      100 |    89.47 |
  Governance.sol      |    93.33 |    83.33 |      100 |    93.75 |
----------------------|----------|----------|----------|----------|
All files             |    94.12 |    85.71 |      100 |    94.44 |
----------------------|----------|----------|----------|----------|
```

### Foundry Coverage

```bash
# Базовый запуск
forge coverage

# С детальным отчётом
forge coverage --report summary

# Генерация lcov отчёта
forge coverage --report lcov

# Фильтрация по контракту
forge coverage --match-contract Token
```

**Пример вывода:**

```
| File                | % Lines         | % Statements    | % Branches      | % Funcs        |
|---------------------|-----------------|-----------------|-----------------|----------------|
| src/Token.sol       | 100.00% (25/25) | 100.00% (30/30) | 100.00% (10/10) | 100.00% (8/8)  |
| src/Vault.sol       | 87.50% (14/16)  | 85.71% (18/21)  | 75.00% (6/8)    | 100.00% (5/5)  |
| Total               | 92.86% (39/42)  | 91.84% (45/49)  | 87.50% (14/16)  | 100.00% (13/13)|
```

## Настройка и конфигурация

### Hardhat Coverage конфигурация

```javascript
// .solcover.js
module.exports = {
  // Пропуск файлов из покрытия
  skipFiles: [
    'mocks/',
    'test/',
    'interfaces/',
    'libraries/external/'
  ],

  // Модификаторы, которые нужно пропустить
  mocha: {
    grep: "@skip-on-coverage",
    invert: true
  },

  // Настройки компиляции для coverage
  configureYulOptimizer: true,

  // Изменение лимита газа для coverage
  providerOptions: {
    default_balance_ether: 10000,
  }
};
```

### Foundry Coverage конфигурация

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[profile.coverage]
# Отключаем оптимизации для точного покрытия
optimizer = false
via_ir = false
```

## Анализ отчётов

### HTML отчёт (Hardhat)

```bash
# Генерация
npx hardhat coverage

# Отчёт создаётся в coverage/index.html
open coverage/index.html
```

HTML отчёт показывает:
- Визуальное выделение непокрытых строк
- Детализация по файлам и функциям
- Интерактивная навигация

### LCOV отчёт (для CI/CD)

```bash
# Foundry
forge coverage --report lcov

# Просмотр в VS Code с расширением Coverage Gutters
# или загрузка в Codecov/Coveralls
```

### Интеграция с Codecov

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run coverage
        run: forge coverage --report lcov

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./lcov.info
          fail_ci_if_error: true
```

## Интерпретация результатов

### Что означает низкое покрытие

```
┌─────────────────────────────────────────────────────────────────┐
│              Интерпретация покрытия                              │
├──────────────────┬──────────────────────────────────────────────┤
│ < 50%            │ Критически низкое. Серьёзный риск багов.     │
├──────────────────┼──────────────────────────────────────────────┤
│ 50-70%           │ Недостаточное. Требуется доработка тестов.   │
├──────────────────┼──────────────────────────────────────────────┤
│ 70-85%           │ Хорошее для большинства проектов.            │
├──────────────────┼──────────────────────────────────────────────┤
│ 85-95%           │ Отличное. Стандарт для DeFi.                 │
├──────────────────┼──────────────────────────────────────────────┤
│ > 95%            │ Превосходное. Максимальная уверенность.      │
└──────────────────┴──────────────────────────────────────────────┘
```

### Ловушки высокого покрытия

**100% покрытие не гарантирует отсутствие багов!**

```solidity
contract Vulnerable {
    mapping(address => uint256) public balances;

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");

        // Reentrancy vulnerability!
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        balances[msg.sender] -= amount; // Уменьшаем после отправки
    }
}
```

```javascript
// Тест покрывает 100% строк, но не находит reentrancy
it("should withdraw", async function() {
  await vault.deposit({ value: 100 });
  await vault.withdraw(100);
  expect(await vault.balances(owner)).to.equal(0);
});
```

**Решение:** Комбинируйте coverage с:
- Fuzz testing
- Invariant testing
- Формальной верификацией
- Аудитом

### Важность Branch Coverage

```solidity
function transfer(address to, uint256 amount) external returns (bool) {
    if (to == address(0)) {      // Branch 1
        revert("Zero address");
    }

    if (amount == 0) {            // Branch 2
        return true;              // Early return - легко пропустить
    }

    if (balances[msg.sender] < amount) {  // Branch 3
        revert("Insufficient");
    }

    balances[msg.sender] -= amount;
    balances[to] += amount;

    return true;
}
```

**Тесты для 100% branch coverage:**

```javascript
describe("Transfer branches", function() {
  it("reverts on zero address", async function() {
    await expect(token.transfer(ethers.ZeroAddress, 100))
      .to.be.revertedWith("Zero address");
  });

  it("handles zero amount", async function() {
    const result = await token.transfer(addr1.address, 0);
    expect(result).to.be.true;
  });

  it("reverts on insufficient balance", async function() {
    await expect(token.connect(addr1).transfer(addr2.address, 100))
      .to.be.revertedWith("Insufficient");
  });

  it("transfers successfully", async function() {
    await token.transfer(addr1.address, 100);
    expect(await token.balanceOf(addr1.address)).to.equal(100);
  });
});
```

## Улучшение покрытия

### Идентификация непокрытого кода

```bash
# Hardhat — смотрим HTML отчёт
# Красные строки = непокрытые
open coverage/index.html

# Foundry — детальный отчёт
forge coverage --report debug
```

### Стратегии улучшения

#### 1. Тестирование всех модификаторов

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

modifier whenNotPaused() {
    require(!paused, "Paused");
    _;
}
```

```javascript
describe("Modifiers", function() {
  it("reverts when not owner", async function() {
    await expect(contract.connect(addr1).adminFunction())
      .to.be.revertedWith("Not owner");
  });

  it("reverts when paused", async function() {
    await contract.pause();
    await expect(contract.someFunction())
      .to.be.revertedWith("Paused");
  });
});
```

#### 2. Тестирование событий

```javascript
it("emits Transfer event", async function() {
  await expect(token.transfer(addr1.address, 100))
    .to.emit(token, "Transfer")
    .withArgs(owner.address, addr1.address, 100);
});
```

#### 3. Тестирование require/revert

```javascript
// Все условия require должны быть протестированы
it("should cover all require statements", async function() {
  // require 1: amount > 0
  await expect(vault.deposit(0))
    .to.be.revertedWith("Amount must be positive");

  // require 2: sufficient allowance
  await expect(vault.depositToken(token.address, 100))
    .to.be.revertedWith("Insufficient allowance");

  // require 3: sufficient balance
  await token.approve(vault.address, ethers.MaxUint256);
  await expect(vault.depositToken(token.address, ethers.MaxUint256))
    .to.be.revertedWith("Insufficient balance");
});
```

#### 4. Edge cases и граничные значения

```javascript
describe("Edge cases", function() {
  it("handles maximum values", async function() {
    const maxUint = ethers.MaxUint256;
    await expect(token.transfer(addr1, maxUint))
      .to.be.revertedWith("Insufficient balance");
  });

  it("handles minimum values", async function() {
    await token.transfer(addr1, 1); // Минимальный перевод
    expect(await token.balanceOf(addr1)).to.equal(1);
  });

  it("handles empty arrays", async function() {
    await expect(batch.processItems([]))
      .to.be.revertedWith("Empty array");
  });
});
```

## Coverage в CI/CD

### GitHub Actions пример

```yaml
name: Coverage

on:
  push:
    branches: [main]
  pull_request:

jobs:
  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run coverage
        run: npx hardhat coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 85" | bc -l) )); then
            echo "Coverage is below 85%: $COVERAGE%"
            exit 1
          fi

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/
```

### Pre-commit hook

```bash
#!/bin/sh
# .husky/pre-commit

# Быстрая проверка coverage перед коммитом
npx hardhat coverage --testfiles "test/unit/**/*.test.js"

COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
if (( $(echo "$COVERAGE < 80" | bc -l) )); then
  echo "Coverage dropped below 80%. Please add more tests."
  exit 1
fi
```

## Best Practices

### 1. Устанавливайте минимальные пороги

```javascript
// package.json
{
  "scripts": {
    "coverage": "hardhat coverage",
    "coverage:check": "hardhat coverage && node scripts/check-coverage.js"
  }
}
```

```javascript
// scripts/check-coverage.js
const coverage = require('./coverage/coverage-summary.json');

const thresholds = {
  lines: 85,
  branches: 80,
  functions: 100,
  statements: 85
};

let failed = false;

for (const [metric, threshold] of Object.entries(thresholds)) {
  const actual = coverage.total[metric].pct;
  if (actual < threshold) {
    console.error(`${metric} coverage ${actual}% is below threshold ${threshold}%`);
    failed = true;
  }
}

if (failed) process.exit(1);
console.log('Coverage thresholds passed!');
```

### 2. Отслеживайте тренды

- Интегрируйте с Codecov или Coveralls
- Блокируйте PR при падении покрытия
- Визуализируйте историю в дашборде

### 3. Не гонитесь за 100%

```javascript
// Некоторый код сложно или бессмысленно покрывать
// Используйте istanbul ignore для осознанного исключения

/* istanbul ignore next */
function emergencyWithdraw() external onlyOwner {
    // Аварийный вывод — тестируется вручную
    selfdestruct(payable(owner));
}
```

### 4. Комбинируйте с другими метриками

```
Комплексный анализ качества:
├── Code Coverage (85%+)
├── Mutation Testing Score
├── Gas Consumption Report
├── Static Analysis (Slither)
└── Security Audit Results
```

## Заключение

Code coverage — важный, но не единственный показатель качества тестов. Высокое покрытие даёт уверенность, что код выполняется в тестах, но не гарантирует отсутствие логических ошибок или уязвимостей. Используйте coverage как часть комплексного подхода к тестированию, включающего fuzz-тестирование, инвариантные тесты и аудит безопасности.
