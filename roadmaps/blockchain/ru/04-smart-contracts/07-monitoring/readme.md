# Мониторинг смарт-контрактов

## Введение

Мониторинг смарт-контрактов — это критически важная часть жизненного цикла DeFi-приложений и блокчейн-проектов. После деплоя контракта на mainnet необходимо постоянно отслеживать его состояние, активность пользователей и потенциальные угрозы безопасности.

## Зачем мониторить смарт-контракты

### Основные причины

1. **Безопасность** — раннее обнаружение подозрительной активности и атак
2. **Отладка** — выявление ошибок в продакшене
3. **Бизнес-аналитика** — понимание поведения пользователей
4. **Управление газом** — оптимизация стоимости транзакций
5. **Compliance** — соответствие регуляторным требованиям
6. **SLA** — обеспечение доступности и производительности

### Что нужно мониторить

- Вызовы функций контракта
- Эмитируемые события (events)
- Изменения состояния (storage)
- Баланс контракта
- Газ и стоимость транзакций
- Неудачные транзакции (reverts)
- Взаимодействия с другими контрактами

## События (Events) и их отслеживание

### Основы событий в Solidity

События — это механизм логирования в Ethereum, который позволяет записывать данные в журнал транзакций:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TokenVault {
    // Объявление событий
    event Deposit(
        address indexed user,
        uint256 amount,
        uint256 timestamp
    );

    event Withdrawal(
        address indexed user,
        uint256 indexed tokenId,
        uint256 amount
    );

    event EmergencyStop(
        address indexed admin,
        string reason
    );

    // indexed параметры можно фильтровать (макс. 3)
    event Transfer(
        address indexed from,
        address indexed to,
        uint256 indexed tokenId,
        uint256 amount,
        bytes data
    );

    function deposit() external payable {
        // Эмитируем событие при депозите
        emit Deposit(msg.sender, msg.value, block.timestamp);
    }
}
```

### Подписка на события с ethers.js

```javascript
const { ethers } = require('ethers');

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const contract = new ethers.Contract(contractAddress, abi, provider);

// Подписка на конкретное событие
contract.on("Deposit", (user, amount, timestamp, event) => {
    console.log(`Deposit: ${user} deposited ${ethers.formatEther(amount)} ETH`);
    console.log(`Block: ${event.log.blockNumber}`);
    console.log(`TX Hash: ${event.log.transactionHash}`);
});

// Подписка с фильтром по indexed параметру
const filter = contract.filters.Transfer(
    "0x1234...",  // from (indexed)
    null,          // to (любой)
    null           // tokenId (любой)
);

contract.on(filter, (from, to, tokenId, amount, data, event) => {
    console.log(`Transfer from ${from} to ${to}`);
});

// Запрос исторических событий
async function getHistoricalEvents() {
    const fromBlock = 18000000;
    const toBlock = "latest";

    const events = await contract.queryFilter(
        contract.filters.Deposit(),
        fromBlock,
        toBlock
    );

    for (const event of events) {
        console.log(`Block ${event.blockNumber}: ${event.args.user} - ${event.args.amount}`);
    }
}
```

## Инструменты мониторинга

### The Graph

The Graph — это децентрализованный протокол для индексации и запроса данных блокчейна с помощью GraphQL.

**Создание субграфа (subgraph.yaml):**

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: TokenVault
    network: mainnet
    source:
      address: "0x1234567890123456789012345678901234567890"
      abi: TokenVault
      startBlock: 18000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Deposit
        - Withdrawal
        - User
      abis:
        - name: TokenVault
          file: ./abis/TokenVault.json
      eventHandlers:
        - event: Deposit(indexed address,uint256,uint256)
          handler: handleDeposit
        - event: Withdrawal(indexed address,indexed uint256,uint256)
          handler: handleWithdrawal
      file: ./src/mapping.ts
```

**GraphQL схема (schema.graphql):**

```graphql
type User @entity {
  id: Bytes!
  totalDeposited: BigInt!
  totalWithdrawn: BigInt!
  deposits: [Deposit!]! @derivedFrom(field: "user")
  withdrawals: [Withdrawal!]! @derivedFrom(field: "user")
  lastActivity: BigInt!
}

type Deposit @entity {
  id: Bytes!
  user: User!
  amount: BigInt!
  timestamp: BigInt!
  blockNumber: BigInt!
  transactionHash: Bytes!
}

type Withdrawal @entity {
  id: Bytes!
  user: User!
  tokenId: BigInt!
  amount: BigInt!
  timestamp: BigInt!
}

type DailyStats @entity {
  id: ID!
  date: BigInt!
  totalDeposits: BigInt!
  totalWithdrawals: BigInt!
  uniqueUsers: BigInt!
}
```

**Маппинг (mapping.ts):**

```typescript
import { Deposit as DepositEvent, Withdrawal as WithdrawalEvent } from "../generated/TokenVault/TokenVault";
import { Deposit, Withdrawal, User, DailyStats } from "../generated/schema";
import { BigInt, Bytes } from "@graphprotocol/graph-ts";

export function handleDeposit(event: DepositEvent): void {
    // Создаем или обновляем пользователя
    let user = User.load(event.params.user);
    if (user == null) {
        user = new User(event.params.user);
        user.totalDeposited = BigInt.fromI32(0);
        user.totalWithdrawn = BigInt.fromI32(0);
    }
    user.totalDeposited = user.totalDeposited.plus(event.params.amount);
    user.lastActivity = event.block.timestamp;
    user.save();

    // Создаем запись депозита
    let deposit = new Deposit(event.transaction.hash.concatI32(event.logIndex.toI32()));
    deposit.user = user.id;
    deposit.amount = event.params.amount;
    deposit.timestamp = event.params.timestamp;
    deposit.blockNumber = event.block.number;
    deposit.transactionHash = event.transaction.hash;
    deposit.save();

    // Обновляем дневную статистику
    let dayId = event.block.timestamp.toI32() / 86400;
    let stats = DailyStats.load(dayId.toString());
    if (stats == null) {
        stats = new DailyStats(dayId.toString());
        stats.date = BigInt.fromI32(dayId * 86400);
        stats.totalDeposits = BigInt.fromI32(0);
        stats.totalWithdrawals = BigInt.fromI32(0);
        stats.uniqueUsers = BigInt.fromI32(0);
    }
    stats.totalDeposits = stats.totalDeposits.plus(event.params.amount);
    stats.save();
}
```

### Tenderly

Tenderly — это платформа для мониторинга, отладки и симуляции смарт-контрактов.

**Настройка алертов через Tenderly API:**

```javascript
const axios = require('axios');

const TENDERLY_API_KEY = process.env.TENDERLY_API_KEY;
const ACCOUNT_SLUG = 'your-account';
const PROJECT_SLUG = 'your-project';

async function createAlert() {
    const alert = {
        name: "Large Deposit Alert",
        description: "Alert when deposit exceeds 100 ETH",
        network: 1, // mainnet
        type: "event",
        alert_type: "event_emitted",
        target: {
            address: "0x1234567890123456789012345678901234567890",
            event_name: "Deposit"
        },
        condition: {
            type: "greater_than",
            parameter_name: "amount",
            value: "100000000000000000000" // 100 ETH in wei
        },
        notification_channels: [
            {
                type: "webhook",
                url: "https://your-server.com/webhook/tenderly"
            },
            {
                type: "slack",
                webhook_url: "https://hooks.slack.com/services/xxx"
            }
        ]
    };

    const response = await axios.post(
        `https://api.tenderly.co/api/v1/account/${ACCOUNT_SLUG}/project/${PROJECT_SLUG}/alerts`,
        alert,
        {
            headers: {
                'X-Access-Key': TENDERLY_API_KEY,
                'Content-Type': 'application/json'
            }
        }
    );

    console.log('Alert created:', response.data);
}
```

### OpenZeppelin Defender

OpenZeppelin Defender — это платформа безопасности для управления и мониторинга смарт-контрактов.

**Настройка Sentinel (мониторинг):**

```javascript
// defender-config.js
const { Defender } = require('@openzeppelin/defender-sdk');

const client = new Defender({
    apiKey: process.env.DEFENDER_API_KEY,
    apiSecret: process.env.DEFENDER_API_SECRET
});

async function createMonitor() {
    const monitor = await client.monitor.create({
        name: 'Token Vault Monitor',
        type: 'BLOCK',
        network: 'mainnet',
        addresses: ['0x1234567890123456789012345678901234567890'],
        abi: contractABI,
        paused: false,
        eventConditions: [
            {
                eventSignature: 'Deposit(address,uint256,uint256)',
                expression: 'amount > 50000000000000000000' // > 50 ETH
            },
            {
                eventSignature: 'EmergencyStop(address,string)',
                expression: null // любой вызов
            }
        ],
        functionConditions: [
            {
                functionSignature: 'withdraw(uint256)',
                expression: 'amount > 100000000000000000000' // > 100 ETH
            }
        ],
        alertThreshold: {
            amount: 1,
            windowSeconds: 3600
        },
        notificationChannels: ['email-channel-id', 'slack-channel-id']
    });

    console.log('Monitor created:', monitor.monitorId);
}
```

## Алерты и уведомления

### Система алертов

```javascript
// alert-system.js
const nodemailer = require('nodemailer');
const axios = require('axios');

class AlertManager {
    constructor() {
        this.transporter = nodemailer.createTransport({
            service: 'gmail',
            auth: {
                user: process.env.EMAIL_USER,
                pass: process.env.EMAIL_PASS
            }
        });
    }

    async sendEmailAlert(subject, body) {
        await this.transporter.sendMail({
            from: process.env.EMAIL_USER,
            to: process.env.ALERT_EMAIL,
            subject: `[ALERT] ${subject}`,
            html: body
        });
    }

    async sendSlackAlert(message, severity = 'warning') {
        const colors = {
            info: '#36a64f',
            warning: '#ff9800',
            critical: '#ff0000'
        };

        await axios.post(process.env.SLACK_WEBHOOK_URL, {
            attachments: [{
                color: colors[severity],
                title: 'Smart Contract Alert',
                text: message,
                ts: Math.floor(Date.now() / 1000)
            }]
        });
    }

    async sendTelegramAlert(message) {
        await axios.post(
            `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendMessage`,
            {
                chat_id: process.env.TELEGRAM_CHAT_ID,
                text: message,
                parse_mode: 'HTML'
            }
        );
    }

    async sendPagerDutyAlert(summary, severity = 'warning') {
        await axios.post('https://events.pagerduty.com/v2/enqueue', {
            routing_key: process.env.PAGERDUTY_KEY,
            event_action: 'trigger',
            payload: {
                summary,
                severity,
                source: 'smart-contract-monitor'
            }
        });
    }
}
```

## On-chain аналитика

### Запросы через The Graph

```graphql
# Топ пользователей по объему депозитов
query TopDepositors {
  users(
    first: 10
    orderBy: totalDeposited
    orderDirection: desc
  ) {
    id
    totalDeposited
    totalWithdrawn
    deposits(first: 5, orderBy: timestamp, orderDirection: desc) {
      amount
      timestamp
    }
  }
}

# Статистика за последние 7 дней
query WeeklyStats {
  dailyStats(
    first: 7
    orderBy: date
    orderDirection: desc
  ) {
    date
    totalDeposits
    totalWithdrawals
    uniqueUsers
  }
}

# Крупные транзакции
query LargeTransactions($minAmount: BigInt!) {
  deposits(
    where: { amount_gte: $minAmount }
    orderBy: timestamp
    orderDirection: desc
    first: 100
  ) {
    user {
      id
    }
    amount
    timestamp
    transactionHash
  }
}
```

## Мониторинг газа и стоимости транзакций

```javascript
// gas-monitor.js
const { ethers } = require('ethers');

class GasMonitor {
    constructor(provider) {
        this.provider = provider;
        this.gasHistory = [];
    }

    async getCurrentGasPrice() {
        const feeData = await this.provider.getFeeData();
        return {
            gasPrice: feeData.gasPrice,
            maxFeePerGas: feeData.maxFeePerGas,
            maxPriorityFeePerGas: feeData.maxPriorityFeePerGas
        };
    }

    async estimateTransactionCost(contract, method, args) {
        const gasEstimate = await contract[method].estimateGas(...args);
        const feeData = await this.provider.getFeeData();

        const costWei = gasEstimate * feeData.gasPrice;
        const costEth = ethers.formatEther(costWei);

        // Получаем цену ETH в USD (например, через Chainlink)
        const ethPriceUsd = await this.getEthPrice();
        const costUsd = parseFloat(costEth) * ethPriceUsd;

        return {
            gasEstimate: gasEstimate.toString(),
            gasPriceGwei: ethers.formatUnits(feeData.gasPrice, 'gwei'),
            costEth,
            costUsd: costUsd.toFixed(2)
        };
    }

    async trackGasUsage(txHash) {
        const receipt = await this.provider.getTransactionReceipt(txHash);
        const tx = await this.provider.getTransaction(txHash);

        const gasUsed = receipt.gasUsed;
        const effectiveGasPrice = receipt.effectiveGasPrice;
        const totalCost = gasUsed * effectiveGasPrice;

        return {
            gasUsed: gasUsed.toString(),
            effectiveGasPrice: ethers.formatUnits(effectiveGasPrice, 'gwei'),
            totalCostEth: ethers.formatEther(totalCost),
            status: receipt.status === 1 ? 'success' : 'failed'
        };
    }
}
```

## Отслеживание безопасности

### Мониторинг подозрительной активности

```javascript
// security-monitor.js
class SecurityMonitor {
    constructor(contract, alertManager) {
        this.contract = contract;
        this.alertManager = alertManager;
        this.knownAddresses = new Set();
        this.recentTransactions = [];
    }

    async checkForAnomalies(event) {
        const checks = [
            this.checkLargeTransaction(event),
            this.checkRapidTransactions(event),
            this.checkNewAddress(event),
            this.checkFlashLoanPattern(event)
        ];

        const results = await Promise.all(checks);
        const anomalies = results.filter(r => r.isAnomaly);

        if (anomalies.length > 0) {
            await this.alertManager.sendSlackAlert(
                `Security anomalies detected:\n${anomalies.map(a => a.message).join('\n')}`,
                'critical'
            );
        }
    }

    checkLargeTransaction(event) {
        const threshold = ethers.parseEther('100');
        const isAnomaly = event.args.amount > threshold;

        return {
            isAnomaly,
            message: isAnomaly
                ? `Large transaction: ${ethers.formatEther(event.args.amount)} ETH from ${event.args.user}`
                : null
        };
    }

    checkRapidTransactions(event) {
        const now = Date.now();
        const windowMs = 60000; // 1 минута
        const threshold = 5; // транзакций

        this.recentTransactions.push({ user: event.args.user, timestamp: now });
        this.recentTransactions = this.recentTransactions.filter(
            t => now - t.timestamp < windowMs
        );

        const userTxCount = this.recentTransactions.filter(
            t => t.user === event.args.user
        ).length;

        const isAnomaly = userTxCount > threshold;

        return {
            isAnomaly,
            message: isAnomaly
                ? `Rapid transactions: ${userTxCount} txs in 1 min from ${event.args.user}`
                : null
        };
    }

    checkNewAddress(event) {
        const isNew = !this.knownAddresses.has(event.args.user);
        this.knownAddresses.add(event.args.user);

        // Новый адрес сам по себе не аномалия, но крупная транзакция от нового — да
        const isAnomaly = isNew && event.args.amount > ethers.parseEther('10');

        return {
            isAnomaly,
            message: isAnomaly
                ? `Large first transaction from new address: ${event.args.user}`
                : null
        };
    }
}
```

## Примеры настройки мониторинга

### Полная настройка мониторинга

```javascript
// monitor-setup.js
const { ethers } = require('ethers');

async function setupMonitoring() {
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    const contract = new ethers.Contract(
        process.env.CONTRACT_ADDRESS,
        contractABI,
        provider
    );

    const alertManager = new AlertManager();
    const securityMonitor = new SecurityMonitor(contract, alertManager);
    const gasMonitor = new GasMonitor(provider);

    // Мониторинг депозитов
    contract.on("Deposit", async (user, amount, timestamp, event) => {
        console.log(`[DEPOSIT] ${user}: ${ethers.formatEther(amount)} ETH`);

        // Проверка безопасности
        await securityMonitor.checkForAnomalies(event);

        // Отслеживание газа
        const gasUsage = await gasMonitor.trackGasUsage(event.log.transactionHash);
        console.log(`Gas used: ${gasUsage.gasUsed}, Cost: ${gasUsage.totalCostEth} ETH`);

        // Алерт для крупных депозитов
        if (amount > ethers.parseEther('50')) {
            await alertManager.sendSlackAlert(
                `Large deposit: ${ethers.formatEther(amount)} ETH from ${user}`,
                'info'
            );
        }
    });

    // Мониторинг экстренных остановок
    contract.on("EmergencyStop", async (admin, reason, event) => {
        await alertManager.sendPagerDutyAlert(
            `EMERGENCY STOP triggered by ${admin}: ${reason}`,
            'critical'
        );

        await alertManager.sendTelegramAlert(
            `<b>CRITICAL ALERT</b>\nEmergency stop activated!\nAdmin: ${admin}\nReason: ${reason}`
        );
    });

    // Мониторинг неудачных транзакций
    provider.on("block", async (blockNumber) => {
        const block = await provider.getBlock(blockNumber, true);

        for (const tx of block.prefetchedTransactions) {
            if (tx.to === process.env.CONTRACT_ADDRESS) {
                const receipt = await provider.getTransactionReceipt(tx.hash);

                if (receipt.status === 0) {
                    console.log(`[FAILED TX] ${tx.hash}`);
                    await alertManager.sendSlackAlert(
                        `Failed transaction: ${tx.hash}`,
                        'warning'
                    );
                }
            }
        }
    });

    console.log('Monitoring started...');
}

setupMonitoring().catch(console.error);
```

## Лучшие практики

1. **Многоуровневые алерты** — используйте разные каналы для разных уровней критичности
2. **Избегайте шума** — настройте пороги, чтобы не получать слишком много уведомлений
3. **Храните историю** — сохраняйте все события для последующего анализа
4. **Тестируйте алерты** — регулярно проверяйте, что уведомления доходят
5. **Документируйте runbook** — опишите действия для каждого типа алерта
6. **Используйте dashboards** — визуализируйте метрики для быстрого анализа
7. **Мониторьте сам мониторинг** — убедитесь, что система мониторинга работает

## Дополнительные ресурсы

- [The Graph Documentation](https://thegraph.com/docs/)
- [Tenderly Documentation](https://docs.tenderly.co/)
- [OpenZeppelin Defender](https://docs.openzeppelin.com/defender/)
- [Ethers.js Events](https://docs.ethers.org/v6/api/contract/#ContractEventName)
