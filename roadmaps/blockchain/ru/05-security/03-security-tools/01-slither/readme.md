# Slither

Slither - это статический анализатор для Solidity от Trail of Bits. Он обнаруживает уязвимости, оптимизирует газ и помогает понять код смарт-контрактов.

## Установка

### Через pip (рекомендуется)

```bash
# Создание виртуального окружения (опционально)
python3 -m venv venv
source venv/bin/activate

# Установка Slither
pip install slither-analyzer

# Проверка установки
slither --version
```

### Через Docker

```bash
docker pull trailofbits/eth-security-toolbox
docker run -it -v $(pwd):/src trailofbits/eth-security-toolbox
```

### Через pipx (изолированная установка)

```bash
pipx install slither-analyzer
```

### Зависимости

Slither требует установленный solc (компилятор Solidity):

```bash
# Установка solc-select для управления версиями
pip install solc-select

# Установка нужной версии
solc-select install 0.8.20
solc-select use 0.8.20
```

## Базовое использование

### Запуск анализа

```bash
# Анализ одного файла
slither Contract.sol

# Анализ Foundry проекта
slither .

# Анализ Hardhat проекта
slither . --hardhat-ignore-compile

# Анализ определенного контракта
slither . --contract MyContract
```

### Пример вывода

```
Contract.sol:15:5: Warning: Reentrancy in Contract.withdraw(uint256) (Contract.sol#15-22):
        External calls:
        - (success,None) = msg.sender.call{value: amount}() (Contract.sol#18)
        State variables written after the call(s):
        - balances[msg.sender] = 0 (Contract.sol#20)
Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#reentrancy-vulnerabilities

Contract.sol:28:9: Warning: Contract locking ether found:
        Contract Contract (Contract.sol#1-50) has payable functions:
         - Contract.deposit() (Contract.sol#10-13)
        But does not have a function to withdraw the ether
Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#contracts-that-lock-ether
```

## Детекторы

Slither имеет более 90 детекторов уязвимостей, разделенных по уровням критичности.

### Уровни критичности

| Уровень | Описание | Примеры |
|---------|----------|---------|
| **High** | Критические уязвимости | Reentrancy, arbitrary send |
| **Medium** | Серьезные проблемы | Dangerous delegatecall, unchecked transfer |
| **Low** | Незначительные проблемы | Missing zero-address check |
| **Informational** | Рекомендации | Naming conventions |
| **Optimization** | Оптимизация газа | State variable packing |

### Список основных детекторов

```bash
# Показать все детекторы
slither --list-detectors

# Показать детекторы по категории
slither --list-detectors-json | jq '.[] | select(.impact == "High")'
```

### Важнейшие детекторы High

| Детектор | Описание |
|----------|----------|
| `reentrancy-eth` | Reentrancy с отправкой ETH |
| `reentrancy-no-eth` | Reentrancy без ETH |
| `arbitrary-send-eth` | Произвольная отправка ETH |
| `controlled-delegatecall` | Контролируемый delegatecall |
| `suicidal` | Незащищенный selfdestruct |
| `unprotected-upgrade` | Незащищенная функция upgrade |

### Фильтрация детекторов

```bash
# Запуск только определенных детекторов
slither . --detect reentrancy-eth,reentrancy-no-eth

# Исключение детекторов
slither . --exclude naming-convention,solc-version

# Фильтр по уровню критичности
slither . --exclude-informational
slither . --exclude-low
slither . --exclude-optimization
```

## Конфигурация

### Файл конфигурации (slither.config.json)

```json
{
  "detectors_to_exclude": [
    "naming-convention",
    "solc-version"
  ],
  "exclude_informational": false,
  "exclude_low": false,
  "exclude_medium": false,
  "exclude_optimization": true,
  "filter_paths": [
    "node_modules",
    "lib",
    "test"
  ],
  "solc_remaps": [
    "@openzeppelin/=node_modules/@openzeppelin/"
  ]
}
```

### Triage Mode (игнорирование ложных срабатываний)

```bash
# Создать базу данных triage
slither . --triage-mode

# При запуске с базой данных известные ложные срабатывания игнорируются
# База хранится в slither.db.json
```

### Пример slither.db.json

```json
[
  {
    "elements": [
      {
        "name": "withdraw",
        "source_mapping": {
          "filename_relative": "src/Vault.sol",
          "lines": [45, 46, 47, 48, 49, 50]
        },
        "type": "function"
      }
    ],
    "description": "Reentrancy in Vault.withdraw...",
    "check": "reentrancy-eth",
    "result": "fp"  // fp = false positive
  }
]
```

## Printers

Printers генерируют информацию о контракте в различных форматах.

### Основные printers

```bash
# Граф наследования
slither . --print inheritance-graph

# Граф вызовов функций
slither . --print call-graph

# Summary контрактов
slither . --print contract-summary

# Переменные состояния
slither . --print vars-and-auth

# Чтение/запись переменных
slither . --print vars-order

# Зависимости между переменными
slither . --print data-dependency

# Human summary
slither . --print human-summary
```

### Пример вывода contract-summary

```
+ Contract Token
  - From Token
    - name() (external)
    - symbol() (external)
    - decimals() (external)
    - totalSupply() (external)
    - balanceOf(address) (external)
    - transfer(address,uint256) (external)
    - allowance(address,address) (external)
    - approve(address,uint256) (external)
    - transferFrom(address,address,uint256) (external)

  State variables:
    - _name (string)
    - _symbol (string)
    - _decimals (uint8)
    - _totalSupply (uint256)
    - _balances (mapping(address => uint256))
    - _allowances (mapping(address => mapping(address => uint256)))
```

### Пример вывода human-summary

```
Compiled with solc
Total number of contracts: 5

+------------------+-------------+---------+------------+--------------+
|       Name       |    Lines    |   SLOC  |   Funcs    |    ERCs      |
+------------------+-------------+---------+------------+--------------+
|      Token       |     125     |    85   |     12     |   ERC20      |
|      Vault       |     200     |   150   |     15     |              |
|      Admin       |      50     |    35   |      5     |              |
+------------------+-------------+---------+------------+--------------+

Number of assembly lines: 15
Number of optimization issues: 8
Number of informational issues: 12
Number of low issues: 3
Number of medium issues: 1
Number of high issues: 0
```

## Продвинутые возможности

### Custom Detectors

Создание собственного детектора:

```python
# my_detector.py
from slither.detectors.abstract_detector import AbstractDetector, DetectorClassification

class MyCustomDetector(AbstractDetector):
    """
    Документация детектора
    """

    ARGUMENT = "my-custom-detector"
    HELP = "Описание детектора"
    IMPACT = DetectorClassification.HIGH
    CONFIDENCE = DetectorClassification.HIGH

    WIKI = "https://example.com/wiki"
    WIKI_TITLE = "Custom Detector Title"
    WIKI_DESCRIPTION = "Detailed description"
    WIKI_EXPLOIT_SCENARIO = "Example exploit scenario"
    WIKI_RECOMMENDATION = "How to fix"

    def _detect(self):
        results = []

        for contract in self.contracts:
            for function in contract.functions:
                # Логика детектирования
                if self._is_vulnerable(function):
                    info = [
                        "Vulnerable function: ",
                        function,
                        "\n"
                    ]
                    res = self.generate_result(info)
                    results.append(res)

        return results

    def _is_vulnerable(self, function):
        # Проверка на уязвимость
        return False
```

Запуск с пользовательским детектором:

```bash
slither . --detect my-custom-detector --plugin my_detector.py
```

### Slither Python API

```python
from slither import Slither

# Загрузка проекта
slither = Slither(".")

# Итерация по контрактам
for contract in slither.contracts:
    print(f"Contract: {contract.name}")

    # Функции
    for function in contract.functions:
        print(f"  Function: {function.name}")
        print(f"    Visibility: {function.visibility}")
        print(f"    Modifiers: {[m.name for m in function.modifiers]}")
        print(f"    State vars read: {[v.name for v in function.state_variables_read]}")
        print(f"    State vars written: {[v.name for v in function.state_variables_written]}")

    # State variables
    for var in contract.state_variables:
        print(f"  State var: {var.name} ({var.type})")

# Анализ наследования
for contract in slither.contracts:
    print(f"{contract.name} inherits from: {[c.name for c in contract.inheritance]}")
```

### Анализ ERC стандартов

```bash
# Проверка соответствия ERC20
slither-check-erc . MyToken ERC20

# Проверка соответствия ERC721
slither-check-erc . MyNFT ERC721

# Проверка соответствия ERC1155
slither-check-erc . MyMultiToken ERC1155
```

### Вывод результатов

```bash
# JSON формат
slither . --json output.json

# SARIF формат (для GitHub Security)
slither . --sarif output.sarif

# Markdown
slither . --markdown output.md

# Checklist формат
slither . --checklist
```

## Интеграция в CI/CD

### GitHub Actions

```yaml
# .github/workflows/slither.yml
name: Slither Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Build contracts
        run: forge build

      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        id: slither
        with:
          sarif: results.sarif
          fail-on: high
          slither-args: --exclude-dependencies

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif
```

### GitLab CI

```yaml
# .gitlab-ci.yml
slither:
  image: trailofbits/eth-security-toolbox
  stage: security
  script:
    - cd /src
    - slither . --json slither-report.json
  artifacts:
    reports:
      sast: slither-report.json
    paths:
      - slither-report.json
  allow_failure: true
  only:
    - merge_requests
    - main
```

### Pre-commit Hook

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
        args:
          - "."
          - "--exclude-dependencies"
          - "--exclude-informational"
          - "--fail-on"
          - "high"
```

## Примеры анализа

### Пример 1: Reentrancy Detection

```solidity
// VulnerableContract.sol
pragma solidity ^0.8.20;

contract Vulnerable {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // VULNERABLE: reentrancy
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");

        // External call before state update
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // State updated after external call
        balances[msg.sender] -= amount;
    }
}
```

Вывод Slither:

```
Reentrancy in Vulnerable.withdraw(uint256) (VulnerableContract.sol#12-21):
        External calls:
        - (success,None) = msg.sender.call{value: amount}() (VulnerableContract.sol#16)
        State variables written after the call(s):
        - balances[msg.sender] -= amount (VulnerableContract.sol#19)
Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#reentrancy-vulnerabilities
```

### Пример 2: Unchecked Return Value

```solidity
// UncheckedTransfer.sol
contract Unsafe {
    IERC20 public token;

    // VULNERABLE: unchecked transfer
    function unsafeTransfer(address to, uint256 amount) external {
        token.transfer(to, amount);  // Return value not checked
    }

    // SAFE: checked transfer
    function safeTransfer(address to, uint256 amount) external {
        require(token.transfer(to, amount), "Transfer failed");
    }
}
```

## Best Practices

### 1. Регулярный запуск

```bash
# Добавить в package.json
{
  "scripts": {
    "slither": "slither . --exclude-dependencies",
    "slither:check": "slither . --exclude-dependencies --fail-on high"
  }
}
```

### 2. Настройка для проекта

Создайте `slither.config.json` с учетом специфики проекта:

```json
{
  "filter_paths": [
    "node_modules",
    "lib",
    "test",
    "script"
  ],
  "exclude_informational": false,
  "exclude_low": false,
  "exclude_optimization": true,
  "detectors_to_exclude": [],
  "solc_remaps": [
    "@openzeppelin/=lib/openzeppelin-contracts/",
    "forge-std/=lib/forge-std/src/"
  ]
}
```

### 3. Обработка результатов

```bash
# Сгенерировать JSON и проанализировать
slither . --json - | jq '.results.detectors[] | select(.impact == "High")'

# Подсчет проблем по уровню
slither . --json - | jq '[.results.detectors[] | .impact] | group_by(.) | map({key: .[0], count: length})'
```

### 4. Интеграция с IDE

Для VS Code установите расширение "Slither" для подсветки проблем прямо в редакторе.

## Ресурсы

- [Официальная документация](https://github.com/crytic/slither)
- [Wiki детекторов](https://github.com/crytic/slither/wiki/Detector-Documentation)
- [Building Secure Contracts](https://github.com/crytic/building-secure-contracts)
- [Trail of Bits Blog](https://blog.trailofbits.com/)
- [Slither API Reference](https://crytic.github.io/slither/slither.html)
