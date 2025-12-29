# Деплой смарт-контрактов

## Введение

Деплой (развертывание) смарт-контракта — это процесс публикации скомпилированного байткода в блокчейн-сеть. После деплоя контракт получает уникальный адрес и становится доступным для взаимодействия. Это критически важный этап, поскольку после размещения в mainnet контракт нельзя изменить.

## Процесс деплоя смарт-контрактов

### Этапы развертывания

1. **Компиляция** — преобразование Solidity-кода в байткод
2. **Подготовка транзакции** — формирование deployment transaction
3. **Подпись транзакции** — использование приватного ключа
4. **Отправка в сеть** — broadcast транзакции в блокчейн
5. **Ожидание подтверждения** — майнинг/валидация блока
6. **Верификация** — публикация исходного кода (опционально)

### Структура deployment транзакции

```javascript
// Deployment транзакция не имеет поля "to"
const deploymentTx = {
    from: "0x...",      // Адрес деплоера
    to: null,           // null для создания контракта
    data: bytecode,     // Скомпилированный байткод
    value: 0,           // ETH для payable конструктора
    gasLimit: 3000000,  // Лимит газа
    gasPrice: "50gwei"  // Цена газа
};
```

## Тестовые сети

### Ethereum Testnets

| Сеть | Консенсус | Фаусет | Использование |
|------|-----------|--------|---------------|
| **Sepolia** | PoS | sepoliafaucet.com | Основная тестовая сеть |
| **Goerli** | PoS | goerlifaucet.com | Устаревает (deprecated) |
| **Holesky** | PoS | holesky-faucet.pk910.de | Для стейкинга |

### Polygon Testnets

| Сеть | Описание | Фаусет |
|------|----------|--------|
| **Mumbai** | Тестнет Polygon | faucet.polygon.technology |
| **Amoy** | Новый тестнет | faucet.polygon.technology |

### Конфигурация сетей в Hardhat

```javascript
// hardhat.config.js
module.exports = {
    networks: {
        sepolia: {
            url: process.env.SEPOLIA_RPC_URL,
            accounts: [process.env.PRIVATE_KEY],
            chainId: 11155111
        },
        mumbai: {
            url: process.env.MUMBAI_RPC_URL,
            accounts: [process.env.PRIVATE_KEY],
            chainId: 80001
        },
        mainnet: {
            url: process.env.MAINNET_RPC_URL,
            accounts: [process.env.PRIVATE_KEY],
            chainId: 1
        }
    }
};
```

## Деплой в Mainnet

### Чек-лист перед деплоем в mainnet

- [ ] Все тесты проходят успешно
- [ ] Аудит безопасности проведен
- [ ] Контракт протестирован в testnet
- [ ] Gas estimation выполнен
- [ ] Приватные ключи защищены
- [ ] Multisig настроен для критических операций
- [ ] План верификации контракта готов

### Оценка стоимости деплоя

```javascript
async function estimateDeploymentCost() {
    const factory = await ethers.getContractFactory("MyContract");
    const deployTx = await factory.getDeployTransaction(constructorArgs);

    const gasEstimate = await ethers.provider.estimateGas(deployTx);
    const gasPrice = await ethers.provider.getGasPrice();

    const cost = gasEstimate.mul(gasPrice);
    console.log(`Estimated cost: ${ethers.utils.formatEther(cost)} ETH`);
}
```

## Hardhat Deploy скрипты

### Базовый скрипт деплоя

```javascript
// scripts/deploy.js
const { ethers, run, network } = require("hardhat");

async function main() {
    console.log("Deploying to network:", network.name);

    const [deployer] = await ethers.getSigners();
    console.log("Deployer address:", deployer.address);
    console.log("Balance:", ethers.utils.formatEther(
        await deployer.getBalance()
    ));

    // Деплой контракта
    const Token = await ethers.getContractFactory("MyToken");
    const token = await Token.deploy("MyToken", "MTK", 1000000);

    await token.deployed();
    console.log("Token deployed to:", token.address);

    // Ждем подтверждений для верификации
    if (network.name !== "hardhat" && network.name !== "localhost") {
        console.log("Waiting for block confirmations...");
        await token.deployTransaction.wait(6);

        // Верификация
        await verify(token.address, ["MyToken", "MTK", 1000000]);
    }
}

async function verify(contractAddress, args) {
    try {
        await run("verify:verify", {
            address: contractAddress,
            constructorArguments: args,
        });
    } catch (e) {
        if (e.message.includes("already verified")) {
            console.log("Contract already verified");
        } else {
            console.error(e);
        }
    }
}

main().catch((error) => {
    console.error(error);
    process.exitCode = 1;
});
```

### Hardhat-deploy плагин

```javascript
// deploy/01_deploy_token.js
module.exports = async ({ getNamedAccounts, deployments }) => {
    const { deploy } = deployments;
    const { deployer } = await getNamedAccounts();

    await deploy("MyToken", {
        from: deployer,
        args: ["MyToken", "MTK", ethers.utils.parseEther("1000000")],
        log: true,
        waitConfirmations: 6,
        autoMine: true, // для локальной сети
    });
};

module.exports.tags = ["MyToken", "all"];
```

## Foundry Deployment

### Скрипт деплоя на Forge

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Script, console} from "forge-std/Script.sol";
import {MyToken} from "../src/MyToken.sol";

contract DeployScript is Script {
    function setUp() public {}

    function run() public returns (MyToken) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);

        MyToken token = new MyToken("MyToken", "MTK", 1000000 ether);

        console.log("Token deployed at:", address(token));

        vm.stopBroadcast();

        return token;
    }
}
```

### Команды деплоя Foundry

```bash
# Деплой в testnet
forge script script/Deploy.s.sol:DeployScript \
    --rpc-url $SEPOLIA_RPC_URL \
    --broadcast \
    --verify \
    --etherscan-api-key $ETHERSCAN_API_KEY

# Симуляция без отправки
forge script script/Deploy.s.sol:DeployScript \
    --rpc-url $SEPOLIA_RPC_URL \
    --fork-url $SEPOLIA_RPC_URL

# С подробным выводом
forge script script/Deploy.s.sol:DeployScript \
    --rpc-url $MAINNET_RPC_URL \
    --broadcast \
    -vvvv
```

## Верификация контрактов

### Etherscan верификация

```bash
# Hardhat
npx hardhat verify --network sepolia DEPLOYED_CONTRACT_ADDRESS "arg1" "arg2"

# Foundry
forge verify-contract \
    --chain-id 11155111 \
    --num-of-optimizations 200 \
    --compiler-version v0.8.19 \
    DEPLOYED_CONTRACT_ADDRESS \
    src/MyToken.sol:MyToken \
    --etherscan-api-key $ETHERSCAN_API_KEY
```

### Программная верификация

```javascript
// Верификация через API
const verifyContract = async (address, sourceCode, contractName) => {
    const params = new URLSearchParams({
        apikey: process.env.ETHERSCAN_API_KEY,
        module: "contract",
        action: "verifysourcecode",
        contractaddress: address,
        sourceCode: sourceCode,
        contractname: contractName,
        compilerversion: "v0.8.19+commit.7dd6d404",
        optimizationUsed: "1",
        runs: "200"
    });

    const response = await fetch(
        "https://api.etherscan.io/api",
        { method: "POST", body: params }
    );

    return response.json();
};
```

## Gas оптимизация при деплое

### Техники снижения gas

```solidity
// 1. Используйте immutable для констант
contract Optimized {
    address public immutable owner;
    uint256 public immutable deployTime;

    constructor() {
        owner = msg.sender;
        deployTime = block.timestamp;
    }
}

// 2. Минимизируйте storage в конструкторе
contract GasEfficient {
    mapping(address => uint256) public balances;

    // Плохо - много записей в storage
    constructor(address[] memory holders, uint256[] memory amounts) {
        for (uint i = 0; i < holders.length; i++) {
            balances[holders[i]] = amounts[i];
        }
    }
}

// 3. Используйте CREATE2 для предсказуемых адресов
contract Factory {
    function deploy(bytes32 salt, bytes memory bytecode)
        external returns (address)
    {
        address addr;
        assembly {
            addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        return addr;
    }
}
```

### Настройки компилятора

```javascript
// hardhat.config.js
module.exports = {
    solidity: {
        version: "0.8.19",
        settings: {
            optimizer: {
                enabled: true,
                runs: 200,  // Меньше runs = дешевле деплой
            },
            viaIR: true,  // Включить IR pipeline для лучшей оптимизации
        }
    }
};
```

## Управление ключами и безопасность

### Переменные окружения

```bash
# .env (НИКОГДА не коммитьте в git!)
PRIVATE_KEY=0x...
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
ETHERSCAN_API_KEY=YOUR_KEY
```

### Использование hardware wallet

```javascript
// Hardhat с Ledger
require("@nomicfoundation/hardhat-ledger");

module.exports = {
    networks: {
        mainnet: {
            url: process.env.MAINNET_RPC_URL,
            ledgerAccounts: [
                "0x..." // Адрес аккаунта Ledger
            ]
        }
    }
};
```

### Multisig деплой

```javascript
// Деплой через Gnosis Safe
const { EthersAdapter, SafeFactory } = require("@safe-global/protocol-kit");

async function deployWithMultisig() {
    const ethAdapter = new EthersAdapter({ ethers, signerOrProvider: signer });

    const safeFactory = await SafeFactory.create({ ethAdapter });

    const safeAccountConfig = {
        owners: ["0x...", "0x...", "0x..."],
        threshold: 2  // 2 из 3 подписей
    };

    const safeSdk = await safeFactory.deploySafe({ safeAccountConfig });
    console.log("Safe deployed:", await safeSdk.getAddress());
}
```

## CI/CD для смарт-контрактов

### GitHub Actions workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Smart Contracts

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      network:
        description: 'Target network'
        required: true
        default: 'sepolia'
        type: choice
        options:
          - sepolia
          - mainnet

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npx hardhat test

      - name: Run coverage
        run: npx hardhat coverage

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3

      - name: Install dependencies
        run: npm ci

      - name: Deploy to ${{ inputs.network }}
        run: npx hardhat run scripts/deploy.js --network ${{ inputs.network }}
        env:
          PRIVATE_KEY: ${{ secrets.DEPLOYER_PRIVATE_KEY }}
          SEPOLIA_RPC_URL: ${{ secrets.SEPOLIA_RPC_URL }}
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
```

### Foundry CI

```yaml
# .github/workflows/foundry.yml
name: Foundry CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Build
        run: forge build

      - name: Test
        run: forge test -vvv

      - name: Gas report
        run: forge test --gas-report
```

## Лучшие практики

1. **Всегда тестируйте в testnet** перед mainnet деплоем
2. **Используйте multisig** для владения критическими контрактами
3. **Верифицируйте исходный код** на Etherscan
4. **Храните deployment артефакты** (адреса, ABI, транзакции)
5. **Документируйте процесс** деплоя для команды
6. **Используйте CI/CD** для автоматизации и консистентности
7. **Проводите аудит** перед mainnet деплоем

## Полезные ресурсы

- [Hardhat Documentation](https://hardhat.org/docs)
- [Foundry Book](https://book.getfoundry.sh/)
- [Etherscan API](https://docs.etherscan.io/)
- [OpenZeppelin Defender](https://defender.openzeppelin.com/)
- [Safe (Gnosis) Documentation](https://docs.safe.global/)
