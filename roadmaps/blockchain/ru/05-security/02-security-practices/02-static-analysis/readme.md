# Статический анализ

Статический анализ — это методика анализа программного кода без его выполнения. Инструменты статического анализа исследуют исходный код или байт-код для обнаружения потенциальных уязвимостей, ошибок и нарушений best practices.

## Что такое статический анализ

```
┌─────────────────────────────────────────────────────────────────┐
│                  Статический vs Динамический анализ             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  СТАТИЧЕСКИЙ АНАЛИЗ              ДИНАМИЧЕСКИЙ АНАЛИЗ            │
│  ─────────────────              ──────────────────              │
│  • Анализ без выполнения        • Требует выполнения кода       │
│  • Быстрый                      • Медленный                     │
│  • Покрывает весь код           • Покрывает выполненные пути    │
│  • Возможны false positives     • Меньше false positives        │
│  • Не требует тестов            • Требует входные данные        │
│                                                                 │
│  Примеры:                       Примеры:                        │
│  Slither, Mythril,              Fuzz-тестирование,              │
│  Securify                       Символьное выполнение           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Типы проверок

### 1. Синтаксические проверки

Проверка корректности синтаксиса и базовых правил языка.

```solidity
// Пример: неиспользуемая переменная
contract Example {
    uint256 unusedVar;  // Warning: unused variable

    function foo() public pure returns (uint256) {
        uint256 x = 5;
        return x;
    }
}
```

### 2. Семантические проверки

Анализ смысла кода и логических ошибок.

```solidity
// Пример: некорректный порядок модификаторов
contract Example {
    function withdraw() external {
        // Проверка после изменения состояния (vulnerable)
        balance[msg.sender] = 0;
        require(balance[msg.sender] > 0);  // Всегда false!
    }
}
```

### 3. Проверки паттернов уязвимостей

Поиск известных уязвимых паттернов.

```
┌─────────────────────────────────────────────────────────────────┐
│              Типичные уязвимости обнаруживаемые SA              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  КРИТИЧЕСКИЕ:                                                   │
│  ├── Reentrancy                                                │
│  ├── Arbitrary external call                                    │
│  ├── Unchecked return values                                   │
│  ├── tx.origin authentication                                   │
│  └── Delegatecall to untrusted contract                        │
│                                                                 │
│  ВЫСОКИЙ РИСК:                                                  │
│  ├── Integer overflow/underflow (до Solidity 0.8)              │
│  ├── Unprotected selfdestruct                                  │
│  ├── Missing access control                                     │
│  └── Block timestamp dependence                                │
│                                                                 │
│  СРЕДНИЙ РИСК:                                                  │
│  ├── DoS с block gas limit                                     │
│  ├── Shadowing state variables                                 │
│  ├── Incorrect equality check                                   │
│  └── Missing zero address validation                           │
│                                                                 │
│  ИНФОРМАЦИОННЫЕ:                                                │
│  ├── Unused variables                                          │
│  ├── Public functions that could be external                   │
│  ├── Missing events                                            │
│  └── Style violations                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Проверки контроля доступа

```solidity
// Пример: отсутствующий контроль доступа
contract Vulnerable {
    address public owner;

    // Любой может вызвать!
    function changeOwner(address newOwner) external {
        owner = newOwner;
    }
}

// Правильно:
contract Secure {
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    function changeOwner(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Zero address");
        owner = newOwner;
    }
}
```

### 5. Data Flow анализ

Отслеживание потока данных через код.

```solidity
// Data flow анализ найдёт:
function vulnerable(address user) external {
    // user приходит извне (tainted data)
    // и используется без проверки
    payable(user).transfer(address(this).balance);
}
```

## Основные инструменты

### Slither

**Slither** — самый популярный статический анализатор для Solidity от Trail of Bits.

#### Установка

```bash
# Через pip
pip install slither-analyzer

# Через pipx (рекомендуется)
pipx install slither-analyzer

# Требует solc
solc-select install 0.8.20
solc-select use 0.8.20
```

#### Использование

```bash
# Базовый анализ
slither .

# Анализ конкретного файла
slither contracts/MyContract.sol

# Указать компилятор
slither . --solc-remaps "@openzeppelin=node_modules/@openzeppelin"

# С Foundry
slither . --foundry-compile-all

# С Hardhat
slither . --hardhat-compile

# Экспорт в JSON
slither . --json report.json

# Конкретные детекторы
slither . --detect reentrancy-eth,arbitrary-send-eth
```

#### Детекторы Slither

```bash
# Посмотреть все детекторы
slither --list-detectors

# Категории детекторов:
# - High: критические уязвимости
# - Medium: серьёзные проблемы
# - Low: незначительные проблемы
# - Informational: стилистические замечания
# - Optimization: оптимизации gas

# Запуск только High и Medium
slither . --filter-paths "node_modules" --exclude-low --exclude-informational
```

#### Примеры обнаруживаемых проблем

```solidity
// 1. REENTRANCY - Slither обнаружит
contract Vulnerable {
    mapping(address => uint) balances;

    function withdraw() external {
        uint amount = balances[msg.sender];
        // Внешний вызов до изменения состояния!
        (bool success,) = msg.sender.call{value: amount}("");
        require(success);
        balances[msg.sender] = 0;
    }
}
// Slither: Reentrancy in Vulnerable.withdraw()

// 2. UNCHECKED TRANSFER - Slither обнаружит
contract Example {
    IERC20 token;

    function deposit(uint amount) external {
        // Возвращаемое значение не проверяется!
        token.transfer(msg.sender, amount);
    }
}
// Slither: Unchecked transfer in Example.deposit()

// 3. TX.ORIGIN - Slither обнаружит
contract Auth {
    address owner;

    function isOwner() internal view returns (bool) {
        return tx.origin == owner;  // Опасно!
    }
}
// Slither: tx.origin used for authentication
```

#### Кастомные детекторы

```python
# my_detector.py
from slither.detectors.abstract_detector import AbstractDetector, DetectorClassification

class MyDetector(AbstractDetector):
    ARGUMENT = "my-detector"
    HELP = "Description of my detector"
    IMPACT = DetectorClassification.HIGH
    CONFIDENCE = DetectorClassification.HIGH

    WIKI = "URL to documentation"
    WIKI_TITLE = "My Detector"
    WIKI_DESCRIPTION = "Detailed description"

    def _detect(self):
        results = []

        for contract in self.compilation_unit.contracts_derived:
            for function in contract.functions:
                # Ваша логика детекции
                if self._is_vulnerable(function):
                    info = [function, " has vulnerability\n"]
                    res = self.generate_result(info)
                    results.append(res)

        return results

    def _is_vulnerable(self, function):
        # Логика проверки
        return False
```

```bash
# Запуск с кастомным детектором
slither . --detect my-detector
```

### Mythril

**Mythril** — инструмент для символьного выполнения и анализа безопасности.

#### Установка

```bash
# Через pip
pip install mythril

# Через Docker
docker pull mythril/myth
```

#### Использование

```bash
# Анализ файла
myth analyze contracts/MyContract.sol

# Анализ развёрнутого контракта
myth analyze --address 0x... --rpc infura

# С увеличенной глубиной анализа
myth analyze contracts/MyContract.sol --execution-timeout 300

# Указать Solidity версию
myth analyze contracts/MyContract.sol --solv 0.8.20

# JSON вывод
myth analyze contracts/MyContract.sol -o json > report.json
```

#### Пример отчёта Mythril

```
==== Integer Arithmetic Bugs ====
SWC ID: 101
Severity: High
Contract: Token
Function name: transfer(address,uint256)
PC: 1234

The arithmetic operator can overflow.
...

==== Reentrancy ====
SWC ID: 107
Severity: High
Contract: Vault
Function name: withdraw()
...
```

### Securify2

**Securify2** — статический анализатор от ETH Zurich.

```bash
# Docker
docker run -v $(pwd):/share securify2 -f /share/contract.sol
```

### Solhint

**Solhint** — линтер для Solidity (стилистические проверки и базовая безопасность).

#### Установка и настройка

```bash
npm install -g solhint

# Инициализация конфигурации
solhint --init
```

#### Конфигурация

```json
// .solhint.json
{
  "extends": "solhint:recommended",
  "plugins": ["security"],
  "rules": {
    "compiler-version": ["error", "^0.8.0"],
    "func-visibility": ["warn", { "ignoreConstructors": true }],
    "max-line-length": ["error", 120],
    "no-inline-assembly": "off",
    "not-rely-on-time": "warn",
    "avoid-low-level-calls": "off",
    "avoid-tx-origin": "error",
    "reason-string": ["warn", { "maxLength": 64 }],
    "security/no-call-value": "error",
    "reentrancy": "error"
  }
}
```

#### Использование

```bash
# Проверка всех файлов
solhint "contracts/**/*.sol"

# Автоисправление
solhint "contracts/**/*.sol" --fix

# Определённые правила
solhint "contracts/**/*.sol" --rule "no-unused-vars: error"
```

### Aderyn

**Aderyn** — современный статический анализатор на Rust от Cyfrin.

```bash
# Установка
cargo install aderyn

# Использование
aderyn .

# С форматом отчёта
aderyn . --output report.md
```

## Интеграция в CI/CD

### GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  slither:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        with:
          target: '.'
          slither-args: '--filter-paths "node_modules|lib" --exclude-informational'
          fail-on: high

  mythril:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Mythril
        uses: mythril/mythril-action@master
        with:
          target: 'contracts/'

  solhint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Solhint
        run: npm install -g solhint

      - name: Run Solhint
        run: solhint 'contracts/**/*.sol'
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - analysis

slither:
  stage: analysis
  image: trailofbits/eth-security-toolbox
  script:
    - cd /src
    - cp -r $CI_PROJECT_DIR/* .
    - npm install
    - slither . --filter-paths "node_modules" --json slither-report.json
  artifacts:
    paths:
      - slither-report.json
    expire_in: 1 week
  only:
    - merge_requests
    - main
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: slither
        name: Slither Static Analysis
        entry: slither
        language: system
        files: \.sol$
        args: ['--filter-paths', 'node_modules', '--exclude-informational']
        pass_filenames: false

      - id: solhint
        name: Solhint Linter
        entry: solhint
        language: node
        files: \.sol$
        args: ['--config', '.solhint.json']
```

### Foundry интеграция

```toml
# foundry.toml
[profile.default]
# ... other settings ...

# Slither через script
[profile.ci.script]
pre_build = "slither . --exclude-informational || true"
```

```bash
# Makefile
.PHONY: analyze

analyze:
	@echo "Running Slither..."
	slither . --filter-paths "node_modules|lib|test" --exclude-informational
	@echo "Running Solhint..."
	solhint 'src/**/*.sol'
```

## Работа с результатами анализа

### Триаж находок

```
┌─────────────────────────────────────────────────────────────────┐
│                    Процесс триажа находок                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ПОЛУЧИТЬ ОТЧЁТ                                              │
│     │                                                           │
│     ▼                                                           │
│  2. КЛАССИФИЦИРОВАТЬ                                            │
│     ├── True Positive (реальная проблема)                      │
│     ├── False Positive (ложное срабатывание)                   │
│     └── Won't Fix (осознанное решение)                         │
│     │                                                           │
│     ▼                                                           │
│  3. ПРИОРИТИЗИРОВАТЬ True Positives                             │
│     ├── Critical → Исправить немедленно                        │
│     ├── High → Исправить до релиза                             │
│     ├── Medium → Запланировать исправление                     │
│     └── Low → По возможности                                    │
│     │                                                           │
│     ▼                                                           │
│  4. ДОКУМЕНТИРОВАТЬ False Positives                             │
│     └── Добавить в исключения с комментарием                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Игнорирование False Positives

```solidity
// Для Slither - inline комментарий
function riskyButIntentional() external {
    // slither-disable-next-line reentrancy-eth
    (bool success,) = msg.sender.call{value: amount}("");
    require(success);
}

// Или на уровне функции
// slither-disable-start reentrancy-eth
function complexFunction() external {
    // ... код с несколькими потенциальными reentrancy
}
// slither-disable-end reentrancy-eth
```

```json
// slither.config.json
{
  "detectors_to_exclude": [
    "naming-convention",
    "solc-version"
  ],
  "exclude_informational": true,
  "exclude_low": false,
  "filter_paths": [
    "node_modules",
    "lib/forge-std"
  ]
}
```

### Формат отчёта

```bash
# Slither JSON отчёт для автоматизации
slither . --json - | jq '.results.detectors[] | select(.impact == "High")'

# Markdown отчёт для review
slither . --checklist > SECURITY_CHECKLIST.md

# SARIF формат для GitHub Security
slither . --sarif report.sarif
```

## Ограничения статического анализа

### Что SA не может найти

1. **Логические ошибки бизнес-логики**
   ```solidity
   // SA не знает что 5% — это ошибка, должно быть 0.5%
   uint256 fee = amount * 5 / 100;  // Технически корректно
   ```

2. **Проблемы в межконтрактных взаимодействиях**
   ```solidity
   // SA не анализирует логику внешнего контракта
   externalProtocol.doSomething(data);
   ```

3. **Oracle manipulation**
   ```solidity
   // SA не знает что oracle может быть манипулирован
   uint256 price = oracle.getPrice();
   ```

4. **Economic attacks**
   ```solidity
   // Flash loan атаки требуют понимания экономики протокола
   ```

### False Positives

Типичные случаи ложных срабатываний:

```solidity
// 1. Reentrancy в view функции
function getBalance() external view returns (uint256) {
    // Slither может пометить как reentrancy, но view не может изменить state
    return externalContract.balance();
}

// 2. Проверка в другой функции
function withdraw() external {
    _checkBalance();  // Slither не видит проверку в другой функции
    payable(msg.sender).transfer(balances[msg.sender]);
    balances[msg.sender] = 0;
}
```

## Комбинирование инструментов

### Рекомендуемый стек

```bash
#!/bin/bash
# security-check.sh

echo "=== Running Security Analysis ==="

echo "[1/4] Solhint (linting)..."
solhint 'contracts/**/*.sol' || exit 1

echo "[2/4] Slither (static analysis)..."
slither . --filter-paths "node_modules|lib" --exclude-informational || exit 1

echo "[3/4] Mythril (symbolic execution)..."
myth analyze contracts/Main.sol --execution-timeout 300 || echo "Mythril warnings found"

echo "[4/4] Aderyn (additional checks)..."
aderyn . || echo "Aderyn warnings found"

echo "=== Analysis Complete ==="
```

### Матрица покрытия

| Уязвимость | Slither | Mythril | Aderyn |
|------------|---------|---------|--------|
| Reentrancy | Да | Да | Да |
| Integer overflow | Да | Да | Да |
| Access control | Частично | Нет | Да |
| tx.origin | Да | Да | Да |
| Unchecked calls | Да | Да | Да |
| Timestamp dependence | Да | Да | Да |
| DoS | Частично | Частично | Да |

## Best Practices

### 1. Запускайте анализ на каждый PR

```yaml
# Минимальный workflow
on: [pull_request]
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: crytic/slither-action@v0.3.0
```

### 2. Фиксируйте версии инструментов

```dockerfile
# Dockerfile для воспроизводимого анализа
FROM python:3.11-slim

RUN pip install \
    slither-analyzer==0.10.0 \
    mythril==0.24.0

WORKDIR /app
```

### 3. Документируйте все исключения

```solidity
/// @dev Reentrancy safe: state updated before external call via CEI pattern
/// @custom:security slither-disable-next-line reentrancy-eth
function safeWithdraw() external nonReentrant {
    // ...
}
```

### 4. Регулярно обновляйте инструменты

```bash
# Проверка обновлений
pip list --outdated | grep -E "slither|mythril"

# Обновление
pip install --upgrade slither-analyzer mythril
```

### 5. Не игнорируйте предупреждения

```
WRONG: "Slither показывает 50 предупреждений, игнорируем"
RIGHT: "Разобрали все 50 предупреждений, 45 — false positives (задокументированы),
        5 — исправлены"
```

## Ресурсы

- [Slither Documentation](https://github.com/crytic/slither/wiki)
- [Mythril Documentation](https://mythril-classic.readthedocs.io/)
- [Solhint Rules](https://protofire.github.io/solhint/docs/rules.html)
- [Aderyn Documentation](https://github.com/Cyfrin/aderyn)
- [SWC Registry](https://swcregistry.io/) — каталог уязвимостей смарт-контрактов
- [Trail of Bits Security Guidelines](https://github.com/crytic/building-secure-contracts)
