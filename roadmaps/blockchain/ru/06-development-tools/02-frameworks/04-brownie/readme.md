# Brownie

## Введение

**Brownie** - это Python-based фреймворк для разработки и тестирования смарт-контрактов на Ethereum. Brownie особенно популярен в DeFi сообществе и был выбором многих крупных проектов, таких как Yearn Finance.

> **Важно:** Brownie находится в режиме поддержки и не получает активной разработки. Для Python-разработчиков рекомендуется рассмотреть Ape Framework или переход на Foundry/Hardhat. Тем не менее, понимание Brownie полезно для работы с существующими проектами.

## Преимущества Brownie

1. **Python экосистема** - использование привычных Python инструментов
2. **pytest интеграция** - мощное тестирование с fixtures
3. **Интерактивная консоль** - IPython REPL для взаимодействия с контрактами
4. **Встроенный fork mainnet** - простое тестирование с реальным состоянием
5. **Coverage и gas profiling** - из коробки
6. **brownie-mix шаблоны** - готовые стартовые проекты

## Установка

### Требования

- Python >= 3.7
- pip или pipx

### Установка через pipx (рекомендуется)

```bash
# Установка pipx (если не установлен)
pip install pipx
pipx ensurepath

# Установка Brownie
pipx install eth-brownie

# Проверка установки
brownie --version
```

### Установка через pip

```bash
pip install eth-brownie
```

### Зависимости

Brownie автоматически устанавливает Ganache CLI. Убедитесь, что Node.js установлен:

```bash
npm install -g ganache-cli
```

## Структура проекта

### Создание проекта

```bash
# Создание пустого проекта
brownie init my_project
cd my_project

# Создание из шаблона (brownie-mix)
brownie bake token
brownie bake aave
brownie bake chainlink-mix
brownie bake uniswap-v2
```

### Структура директорий

```
my_project/
├── contracts/          # Solidity контракты
│   └── Token.sol
├── interfaces/         # Интерфейсы контрактов
├── scripts/            # Скрипты деплоя и взаимодействия
│   └── deploy.py
├── tests/              # Тесты (pytest)
│   └── test_token.py
├── build/              # Скомпилированные артефакты
│   ├── contracts/
│   └── deployments/    # Информация о деплоях
├── reports/            # Отчеты о покрытии и газе
├── brownie-config.yaml # Конфигурация проекта
└── .env                # Переменные окружения
```

## Конфигурация brownie-config.yaml

```yaml
# brownie-config.yaml
project_structure:
  contracts: contracts
  interfaces: interfaces
  scripts: scripts
  tests: tests

# Настройки компилятора
compiler:
  solc:
    version: 0.8.24
    optimizer:
      enabled: true
      runs: 200
    remappings:
      - "@openzeppelin=.brownie/packages/OpenZeppelin/openzeppelin-contracts@4.9.3"

# Зависимости (пакеты)
dependencies:
  - OpenZeppelin/openzeppelin-contracts@4.9.3
  - smartcontractkit/chainlink@1.10.0

# Настройки сетей
networks:
  default: development

  development:
    gas_limit: max
    gas_price: 0

  mainnet-fork:
    fork: mainnet

# Автофетч контрактов (для взаимодействия с mainnet)
autofetch_sources: true

# Dotenv файл
dotenv: .env

# Именованные аккаунты
wallets:
  from_key: ${PRIVATE_KEY}
  from_mnemonic: ${MNEMONIC}
```

## Компиляция

```bash
# Компиляция всех контрактов
brownie compile

# Принудительная перекомпиляция
brownie compile --all
```

## Деплой и скрипты

### Базовый скрипт деплоя

```python
# scripts/deploy.py
from brownie import Token, accounts, network, config

def main():
    # Получение аккаунта
    if network.show_active() == "development":
        account = accounts[0]
    else:
        account = accounts.load("deployer")  # Именованный аккаунт
        # или
        account = accounts.add(config["wallets"]["from_key"])

    print(f"Deploying from: {account.address}")
    print(f"Balance: {account.balance() / 1e18} ETH")

    # Деплой контракта
    token = Token.deploy(
        "MyToken",          # name
        "MTK",              # symbol
        1_000_000 * 10**18, # initial supply
        {"from": account}
    )

    print(f"Token deployed at: {token.address}")
    return token
```

### Запуск скриптов

```bash
# Запуск на локальной сети
brownie run scripts/deploy.py

# На конкретной сети
brownie run scripts/deploy.py --network sepolia

# С форком mainnet
brownie run scripts/deploy.py --network mainnet-fork
```

### Скрипт с взаимодействием

```python
# scripts/interact.py
from brownie import Token, accounts, Contract

def main():
    # Загрузка задеплоенного контракта
    token = Token[-1]  # Последний задеплоенный
    # или
    token = Token.at("0xContractAddress")

    account = accounts[0]

    # Чтение данных
    print(f"Name: {token.name()}")
    print(f"Symbol: {token.symbol()}")
    print(f"Total Supply: {token.totalSupply() / 1e18}")
    print(f"Balance of {account}: {token.balanceOf(account) / 1e18}")

    # Отправка транзакции
    tx = token.transfer(accounts[1], 100 * 10**18, {"from": account})
    tx.wait(1)  # Ждем подтверждения

    print(f"Transaction hash: {tx.txid}")
    print(f"Gas used: {tx.gas_used}")
```

## Тестирование с pytest

### Базовый тест

```python
# tests/test_token.py
import pytest
from brownie import Token, accounts, reverts

@pytest.fixture
def token():
    """Деплой токена для каждого теста"""
    return Token.deploy(
        "MyToken",
        "MTK",
        1_000_000 * 10**18,
        {"from": accounts[0]}
    )

@pytest.fixture
def owner():
    return accounts[0]

@pytest.fixture
def user():
    return accounts[1]

class TestDeployment:
    def test_name(self, token):
        assert token.name() == "MyToken"

    def test_symbol(self, token):
        assert token.symbol() == "MTK"

    def test_initial_supply(self, token, owner):
        assert token.totalSupply() == 1_000_000 * 10**18
        assert token.balanceOf(owner) == 1_000_000 * 10**18

class TestTransfer:
    def test_transfer(self, token, owner, user):
        initial_balance = token.balanceOf(owner)
        amount = 100 * 10**18

        token.transfer(user, amount, {"from": owner})

        assert token.balanceOf(user) == amount
        assert token.balanceOf(owner) == initial_balance - amount

    def test_transfer_insufficient_balance(self, token, user):
        with reverts("ERC20: insufficient balance"):
            token.transfer(accounts[2], 100 * 10**18, {"from": user})

    def test_transfer_event(self, token, owner, user):
        amount = 100 * 10**18
        tx = token.transfer(user, amount, {"from": owner})

        assert len(tx.events) == 1
        assert tx.events["Transfer"]["from"] == owner
        assert tx.events["Transfer"]["to"] == user
        assert tx.events["Transfer"]["value"] == amount

class TestApproval:
    def test_approve_and_transfer_from(self, token, owner, user):
        amount = 100 * 10**18

        token.approve(user, amount, {"from": owner})
        assert token.allowance(owner, user) == amount

        token.transferFrom(owner, accounts[2], amount, {"from": user})
        assert token.balanceOf(accounts[2]) == amount
```

### Фикстуры с разными scope

```python
# tests/conftest.py
import pytest
from brownie import Token, accounts, network

@pytest.fixture(scope="module")
def token():
    """Деплой один раз для всего модуля"""
    return Token.deploy(
        "MyToken",
        "MTK",
        1_000_000 * 10**18,
        {"from": accounts[0]}
    )

@pytest.fixture(autouse=True)
def isolation(fn_isolation):
    """Изоляция каждого теста (откат состояния)"""
    pass

@pytest.fixture
def funded_account():
    """Аккаунт с ETH"""
    account = accounts.add()
    accounts[0].transfer(account, "10 ether")
    return account
```

### Тестирование с mainnet fork

```python
# tests/test_integration.py
import pytest
from brownie import Contract, accounts, chain

# Адреса mainnet контрактов
UNISWAP_ROUTER = "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"
DAI = "0x6B175474E89094C44Da98b954EecscdeCB5BE3F3E"
WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"

@pytest.fixture(scope="module")
def router():
    return Contract.from_explorer(UNISWAP_ROUTER)

@pytest.fixture(scope="module")
def dai():
    return Contract.from_explorer(DAI)

@pytest.fixture
def whale(accounts):
    """Impersonate whale account"""
    whale_address = "0x..." # Адрес с большим балансом DAI
    return accounts.at(whale_address, force=True)

def test_swap_eth_for_dai(router, dai, accounts):
    account = accounts[0]
    initial_dai = dai.balanceOf(account)

    # Swap 1 ETH for DAI
    router.swapExactETHForTokens(
        0,  # min amount out
        [WETH, DAI],
        account,
        chain.time() + 300,
        {"from": account, "value": "1 ether"}
    )

    assert dai.balanceOf(account) > initial_dai
```

### Запуск тестов

```bash
# Запуск всех тестов
brownie test

# Конкретный файл
brownie test tests/test_token.py

# Конкретный тест
brownie test tests/test_token.py::TestTransfer::test_transfer

# С coverage
brownie test --coverage

# С gas profiling
brownie test --gas

# На mainnet fork
brownie test --network mainnet-fork
```

## Интерактивная консоль

Brownie предоставляет мощную IPython консоль:

```bash
# Запуск консоли
brownie console

# На конкретной сети
brownie console --network mainnet-fork
```

```python
# В консоли
>>> from brownie import *

# Деплой контракта
>>> token = Token.deploy("MyToken", "MTK", 1e24, {"from": accounts[0]})

# Взаимодействие
>>> token.name()
'MyToken'

>>> token.transfer(accounts[1], 1000 * 1e18, {"from": accounts[0]})

# Работа с mainnet контрактами
>>> dai = Contract.from_explorer("0x6B175474E89094C44Da98b954EecscdeCB5BE3F3E")
>>> dai.symbol()
'DAI'

# История транзакций
>>> history[-1].info()

# Манипуляция временем (только на development)
>>> chain.sleep(3600)  # +1 час
>>> chain.mine(10)     # +10 блоков

# Snapshot и revert
>>> snapshot_id = chain.snapshot()
>>> # ... действия ...
>>> chain.revert(snapshot_id)
```

## Работа с mainnet fork

### Настройка в brownie-config.yaml

```yaml
networks:
  mainnet-fork:
    fork: https://mainnet.infura.io/v3/${WEB3_INFURA_PROJECT_ID}
    # или с Alchemy
    fork: https://eth-mainnet.alchemyapi.io/v2/${ALCHEMY_API_KEY}
```

### Impersonation аккаунтов

```python
from brownie import accounts

# Получение контроля над любым адресом
whale = accounts.at("0xWhaleAddress", force=True)

# Теперь можно отправлять транзакции от имени whale
token.transfer(accounts[0], 1000 * 1e18, {"from": whale})
```

### Форк на конкретном блоке

```bash
brownie console --network mainnet-fork --fork-block 18500000
```

## brownie-mix шаблоны

Готовые стартовые проекты:

```bash
# Список доступных шаблонов
brownie bake --list

# Популярные шаблоны
brownie bake token         # ERC20 токен
brownie bake nft-mix       # NFT проект
brownie bake aave          # Интеграция с Aave
brownie bake chainlink-mix # Chainlink оракулы
brownie bake uniswap-v2    # Uniswap интеграция
brownie bake vyper         # Vyper контракты
```

## Работа с аккаунтами

```python
from brownie import accounts

# Локальные аккаунты (только development)
account = accounts[0]

# Создание нового аккаунта
new_account = accounts.add()
print(new_account.private_key)

# Загрузка из приватного ключа
account = accounts.add("0x...")

# Загрузка из мнемоники
account = accounts.from_mnemonic("word1 word2 ...")

# Именованные аккаунты (сохранение зашифрованно)
accounts.add("0x...", id="deployer")
accounts.load("deployer")  # Запросит пароль
```

## Gas profiling

```python
# tests/test_gas.py
def test_transfer_gas(token, owner, user):
    tx = token.transfer(user, 100 * 10**18, {"from": owner})

    # Информация о газе
    print(f"Gas used: {tx.gas_used}")
    print(f"Gas price: {tx.gas_price}")
    print(f"Total cost: {tx.gas_used * tx.gas_price / 1e18} ETH")
```

```bash
# Запуск с отчетом о газе
brownie test --gas
```

## Coverage

```bash
# Запуск с coverage
brownie test --coverage

# GUI отчет
brownie gui  # Откроет в браузере
```

## Верификация контрактов

```python
# scripts/verify.py
from brownie import Token, network

def main():
    token = Token[-1]  # Последний задеплоенный

    Token.publish_source(token)
```

```bash
export ETHERSCAN_TOKEN=your_api_key
brownie run scripts/verify.py --network sepolia
```

## Статус проекта

> **Важно:** Brownie находится в режиме поддержки с 2023 года. Основной разработчик (iamdefinitelyahuman) перешел на другие проекты.

**Что это означает:**
- Критические баги исправляются
- Новые функции не добавляются
- Поддержка новых версий Solidity может отставать
- Сообщество постепенно мигрирует

**Альтернативы для Python разработчиков:**
1. **Ape Framework** - духовный наследник Brownie
2. **Foundry** - если готовы перейти на Solidity тесты
3. **Hardhat** - если готовы перейти на TypeScript

## Миграция на Ape Framework

Ape Framework создан теми же разработчиками и имеет похожий API:

```bash
# Установка Ape
pipx install eth-ape

# Плагины
ape plugins install solidity
ape plugins install alchemy
```

```python
# Ape имеет похожий синтаксис
from ape import accounts, project

def main():
    account = accounts.load("deployer")
    token = project.Token.deploy("MyToken", "MTK", 1e24, sender=account)
```

## Лучшие практики

1. **Используйте фикстуры** - для переиспользования setup
2. **fn_isolation** - изолируйте тесты друг от друга
3. **Именованные аккаунты** - безопасное хранение ключей
4. **mainnet fork** - тестирование интеграций
5. **Coverage** - отслеживайте покрытие тестами

## Ресурсы

- [Документация Brownie](https://eth-brownie.readthedocs.io/)
- [Brownie GitHub](https://github.com/eth-brownie/brownie)
- [brownie-mix репозиторий](https://github.com/brownie-mix)
- [Ape Framework](https://apeworx.io/) - альтернатива
