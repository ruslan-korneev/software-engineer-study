# Контроль версий для блокчейн-проектов

Контроль версий критически важен в блокчейн-разработке, где ошибки могут привести к потере средств. Правильная организация репозитория, управление секретами и CI/CD процессы обеспечивают безопасность и качество кода.

## 1. Git для блокчейн-проектов

### Особенности работы с Git в блокчейн-разработке

Блокчейн-разработка имеет специфические требования к контролю версий:

1. **Иммутабельность контрактов** - после деплоя код нельзя изменить, поэтому каждый коммит должен быть тщательно проверен
2. **Аудируемость** - история изменений должна быть прозрачной для аудиторов
3. **Версионирование** - важно отслеживать версии контрактов для разных сетей
4. **Безопасность** - необходимо исключить утечку приватных ключей и конфиденциальных данных

### Типичная структура репозитория смарт-контрактов

```
my-defi-project/
├── contracts/              # Исходный код смарт-контрактов
│   ├── core/               # Основная логика
│   │   ├── Token.sol
│   │   └── Vault.sol
│   ├── interfaces/         # Интерфейсы
│   │   └── IToken.sol
│   ├── libraries/          # Библиотеки
│   │   └── SafeMath.sol
│   └── mocks/              # Моки для тестов
│       └── MockOracle.sol
├── scripts/                # Скрипты деплоя
│   ├── deploy.js
│   └── verify.js
├── test/                   # Тесты
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── deployments/            # Адреса задеплоенных контрактов
│   ├── mainnet/
│   ├── goerli/
│   └── sepolia/
├── docs/                   # Документация
├── .github/                # GitHub Actions workflows
│   └── workflows/
├── .env.example            # Пример переменных окружения
├── .gitignore
├── hardhat.config.js       # Конфигурация Hardhat
├── foundry.toml            # Конфигурация Foundry
├── package.json
└── README.md
```

## 2. .gitignore для смарт-контрактов

### Что нужно игнорировать

Основные категории файлов для исключения:

1. **Зависимости** - node_modules, lib (Foundry)
2. **Артефакты компиляции** - artifacts, cache, out
3. **Данные деплоя** - broadcast (Foundry)
4. **Конфиденциальные данные** - .env, приватные ключи
5. **IDE файлы** - .vscode, .idea

### Пример .gitignore для Hardhat

```gitignore
# Dependencies
node_modules/

# Hardhat artifacts
artifacts/
cache/
typechain/
typechain-types/

# Coverage
coverage/
coverage.json

# Environment variables
.env
.env.local
.env.*.local

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*

# Local deployment data (optional)
deployments/localhost/
```

### Пример .gitignore для Foundry

```gitignore
# Foundry artifacts
out/
cache/
broadcast/

# Dependencies
lib/

# Environment
.env
.env.*

# Coverage
lcov.info

# IDE
.idea/
.vscode/

# OS
.DS_Store
```

### Пример .gitignore для Truffle

```gitignore
# Dependencies
node_modules/

# Truffle build
build/

# Environment
.env

# IDE
.idea/
.vscode/

# Coverage
coverage/
coverage.json
```

### Что категорически НЕЛЬЗЯ коммитить

```bash
# НИКОГДА не коммитьте:
.env                    # Содержит приватные ключи и API ключи
*.pem                   # Приватные ключи
*.key                   # Ключи
mnemonic.txt            # Seed-фразы
private_key.txt         # Приватные ключи в текстовом формате
secrets/                # Папка с секретами
```

**Проверка перед коммитом:**

```bash
# Проверить, что секреты не попали в индекс
git diff --cached --name-only | grep -E '\.(env|pem|key)$'

# Поиск приватных ключей в истории
git log -p | grep -E '0x[a-fA-F0-9]{64}'
```

## 3. Управление секретами

### .env файлы и dotenv

Создайте файл `.env.example` для документации:

```bash
# .env.example - коммитится в репозиторий
PRIVATE_KEY=your_private_key_here
INFURA_API_KEY=your_infura_key_here
ETHERSCAN_API_KEY=your_etherscan_key_here
ALCHEMY_API_KEY=your_alchemy_key_here
```

Реальный `.env` файл (НЕ коммитить):

```bash
# .env - НЕ коммитить!
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
INFURA_API_KEY=abc123def456
ETHERSCAN_API_KEY=XYZ789
```

Использование в Hardhat:

```javascript
// hardhat.config.js
require('dotenv').config();

module.exports = {
  networks: {
    sepolia: {
      url: `https://sepolia.infura.io/v3/${process.env.INFURA_API_KEY}`,
      accounts: [process.env.PRIVATE_KEY]
    }
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  }
};
```

### Как безопасно хранить приватные ключи

**Уровни безопасности:**

1. **Разработка (низкий риск):**
   - .env файлы для тестовых сетей
   - Используйте отдельные кошельки только для тестов

2. **Staging (средний риск):**
   - Encrypted secrets в CI/CD
   - Отдельные кошельки с ограниченными средствами

3. **Production (высокий риск):**
   - Hardware wallets (Ledger, Trezor)
   - Multi-signature wallets
   - Key Management Services (AWS KMS, HashiCorp Vault)

### Использование Hardware Wallets для деплоя

**Деплой через Ledger с Hardhat:**

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-ledger");

module.exports = {
  networks: {
    mainnet: {
      url: `https://mainnet.infura.io/v3/${process.env.INFURA_API_KEY}`,
      ledgerAccounts: [
        "0x1234567890123456789012345678901234567890"
      ]
    }
  }
};
```

**Деплой через Frame Wallet:**

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    mainnet: {
      url: "http://127.0.0.1:1248", // Frame RPC
    }
  }
};
```

### GitHub Secrets и CI/CD переменные

Добавление секретов в GitHub:

```
Settings -> Secrets and variables -> Actions -> New repository secret
```

Использование в GitHub Actions:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Testnet

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Deploy to Sepolia
        env:
          PRIVATE_KEY: ${{ secrets.DEPLOYER_PRIVATE_KEY }}
          INFURA_API_KEY: ${{ secrets.INFURA_API_KEY }}
        run: npx hardhat run scripts/deploy.js --network sepolia
```

## 4. Branching стратегии

### Git Flow vs Trunk-based для смарт-контрактов

**Git Flow** - рекомендуется для блокчейн-проектов:

```
main          ─────●─────────────●─────────── (production releases)
                   │             │
release       ─────┼──●──────────┤
                   │  │          │
develop       ─●───┼──┼──●───●───┼──●─────── (integration)
               │   │  │  │   │   │  │
feature/token ─┴───┘  │  │   │   │  │
feature/vault ────────┴──┘   │   │  │
hotfix/bug    ───────────────┴───┘  │
feature/stake ──────────────────────┴─────
```

**Преимущества Git Flow для смарт-контрактов:**
- Четкое разделение между разработкой и production
- Возможность тестирования релизов перед деплоем
- Легко откатить изменения до деплоя

**Trunk-based** - для небольших команд с частыми релизами:

```
main ─●──●──●──●──●──●──●──●──●──●─────────
      │     │     │     │     │
      └─────┴─────┴─────┴─────┴── короткоживущие feature branches
```

### Feature branches для новых контрактов

Именование веток:

```bash
# Новые контракты
feature/token-v2
feature/staking-rewards
feature/governance

# Исправления багов
fix/reentrancy-vulnerability
fix/overflow-check

# Улучшения
improvement/gas-optimization
improvement/access-control

# Документация
docs/natspec-update
```

### Версионирование контрактов

**Семантическое версионирование:**

```solidity
// contracts/Token.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title MyToken
/// @author Your Name
/// @notice ERC20 token implementation
/// @dev Version 1.2.0
contract Token {
    string public constant VERSION = "1.2.0";
    // ...
}
```

**Git tags для релизов:**

```bash
# Создание тега
git tag -a v1.2.0 -m "Release v1.2.0: Added staking feature"

# Публикация тега
git push origin v1.2.0

# Просмотр тегов
git tag -l "v*"
```

## 5. CI/CD для блокчейн-проектов

### GitHub Actions для тестирования

**Базовый workflow для Hardhat:**

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Compile contracts
        run: npx hardhat compile

      - name: Run tests
        run: npx hardhat test

      - name: Run coverage
        run: npx hardhat coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

**Workflow для Foundry:**

```yaml
# .github/workflows/foundry.yml
name: Foundry Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run Forge build
        run: forge build

      - name: Run Forge tests
        run: forge test -vvv

      - name: Run Forge coverage
        run: forge coverage --report lcov
```

### Автоматическая проверка gas usage

```yaml
# .github/workflows/gas-report.yml
name: Gas Report

on:
  pull_request:
    branches: [main]

jobs:
  gas-report:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run gas report
        run: REPORT_GAS=true npx hardhat test

      - name: Compare gas usage
        uses: Rubilmax/foundry-gas-diff@v3.16
        with:
          summaryQuantile: 0.9
          sortCriteria: avg,max
          sortOrders: desc,asc
```

**Конфигурация hardhat-gas-reporter:**

```javascript
// hardhat.config.js
require("hardhat-gas-reporter");

module.exports = {
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    outputFile: "gas-report.txt",
    noColors: true,
    excludeContracts: ["Mocks/"]
  }
};
```

### Slither и другие security анализаторы в CI

```yaml
# .github/workflows/security.yml
name: Security Analysis

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  slither:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        id: slither
        with:
          node-version: '20'
          sarif: results.sarif
          fail-on: high

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif

  mythril:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Mythril
        uses: docker://mythril/myth
        with:
          args: analyze contracts/**/*.sol --solc-json mythril.config.json

  aderyn:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Aderyn
        run: cargo install aderyn

      - name: Run Aderyn
        run: aderyn .
```

### Автоматический деплой в testnets

```yaml
# .github/workflows/deploy-testnet.yml
name: Deploy to Testnet

on:
  push:
    tags:
      - 'v*-rc*'  # v1.0.0-rc1, v1.0.0-rc2

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: testnet

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Compile contracts
        run: npx hardhat compile

      - name: Deploy to Sepolia
        env:
          PRIVATE_KEY: ${{ secrets.TESTNET_DEPLOYER_KEY }}
          INFURA_API_KEY: ${{ secrets.INFURA_API_KEY }}
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
        run: |
          npx hardhat run scripts/deploy.js --network sepolia
          npx hardhat verify --network sepolia

      - name: Save deployment artifacts
        uses: actions/upload-artifact@v3
        with:
          name: deployment-${{ github.ref_name }}
          path: deployments/
```

## 6. Code Review для смарт-контрактов

### Что проверять в PR

**Обязательные проверки:**

1. **Безопасность:**
   - Reentrancy атаки
   - Integer overflow/underflow
   - Access control
   - Front-running уязвимости
   - DoS атаки

2. **Логика:**
   - Корректность бизнес-логики
   - Edge cases
   - Обработка ошибок

3. **Gas оптимизация:**
   - Эффективность циклов
   - Storage vs memory
   - Упаковка переменных

4. **Качество кода:**
   - NatSpec комментарии
   - Соответствие style guide
   - Тестовое покрытие

### Security Checklist

```markdown
## Security Review Checklist

### Access Control
- [ ] Функции имеют правильные модификаторы доступа
- [ ] Owner/admin функции защищены
- [ ] Нет открытых initialize() функций

### Reentrancy
- [ ] Используется ReentrancyGuard или checks-effects-interactions
- [ ] Внешние вызовы в конце функций
- [ ] State обновляется до внешних вызовов

### Input Validation
- [ ] Все параметры проверяются
- [ ] Адреса проверяются на != address(0)
- [ ] Массивы проверяются на длину

### Arithmetic
- [ ] Solidity 0.8+ или SafeMath
- [ ] Проверки на деление на ноль
- [ ] Проверки на overflow в unchecked блоках

### External Calls
- [ ] Проверки возвращаемых значений
- [ ] Обработка failed calls
- [ ] Минимизация gas в callbacks

### Events
- [ ] Важные изменения логируются
- [ ] События индексированы правильно

### Tests
- [ ] Unit тесты для всех функций
- [ ] Edge cases покрыты
- [ ] Coverage > 90%
```

### Инструменты для автоматической проверки

**Pull Request шаблон (.github/PULL_REQUEST_TEMPLATE.md):**

```markdown
## Description
<!-- Describe your changes -->

## Type of change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] I have run `slither` and addressed any issues

## Security Considerations
<!-- Describe any security implications of this change -->

## Gas Impact
<!-- Note any significant changes to gas consumption -->
```

**Автоматические проверки в PR:**

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run format:check

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run coverage
      - uses: codecov/codecov-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: crytic/slither-action@v0.3.0
```

## Заключение

Правильная организация контроля версий в блокчейн-проектах критически важна для безопасности и качества кода. Ключевые принципы:

1. **Никогда не коммитьте секреты** - используйте .env файлы и CI/CD secrets
2. **Используйте Git Flow** - для четкого контроля релизов
3. **Автоматизируйте проверки** - CI/CD должен включать тесты, coverage и security анализ
4. **Проводите тщательный code review** - используйте чеклисты и автоматические инструменты
5. **Версионируйте контракты** - используйте семантическое версионирование и git tags
