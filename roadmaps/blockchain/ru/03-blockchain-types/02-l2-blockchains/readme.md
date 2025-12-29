# L2 блокчейны (Layer 2 Solutions)

## Что такое Layer 2?

**Layer 2 (L2)** — это решения масштабирования, которые работают поверх основного блокчейна (Layer 1), обрабатывая транзакции за его пределами, но при этом наследуя его безопасность. L2 решения позволяют значительно увеличить пропускную способность сети и снизить комиссии.

### Почему L2 необходимы?

Ethereum, как основной L1 для DeFi и NFT, сталкивается с серьёзными ограничениями:

| Проблема L1 | Решение L2 |
|-------------|------------|
| ~15-30 TPS | Тысячи TPS |
| Высокие комиссии ($5-100+) | Комиссии $0.01-0.5 |
| Медленные подтверждения | Мгновенные транзакции |
| Перегрузка сети | Разгрузка основной сети |

## Типы L2 решений

### Rollups

**Rollups** — основной тип L2 решений, который "сворачивает" множество транзакций в один пакет и отправляет его на L1.

```
┌─────────────────────────────────────────────┐
│                Layer 2 (Rollup)              │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
│  │ TX1 │ │ TX2 │ │ TX3 │ │ TX4 │ │ TX5 │   │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘   │
│     └───────┴───────┼───────┴───────┘       │
│                     ▼                        │
│            ┌───────────────┐                 │
│            │  Batch (Пакет) │                │
│            └───────┬───────┘                 │
└────────────────────┼────────────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│           Layer 1 (Ethereum)                 │
│         Compressed Data + Proof              │
└─────────────────────────────────────────────┘
```

### Optimistic Rollups

**Optimistic Rollups** предполагают, что все транзакции валидны по умолчанию ("оптимистичный" подход).

**Как работают:**
1. Секвенсор собирает транзакции и формирует пакет
2. Пакет отправляется на L1 без проверки
3. Есть период оспаривания (~7 дней)
4. Любой может представить **fraud proof** (доказательство мошенничества)
5. Если fraud proof валиден — транзакция откатывается

```javascript
// Упрощённая логика fraud proof
contract OptimisticRollup {
    uint256 public constant CHALLENGE_PERIOD = 7 days;

    struct Batch {
        bytes32 stateRoot;
        uint256 timestamp;
        bool finalized;
    }

    mapping(uint256 => Batch) public batches;

    // Отправка пакета
    function submitBatch(bytes32 stateRoot, bytes calldata txData) external {
        batches[batchCount] = Batch({
            stateRoot: stateRoot,
            timestamp: block.timestamp,
            finalized: false
        });
    }

    // Оспаривание пакета
    function challengeBatch(
        uint256 batchId,
        bytes calldata fraudProof
    ) external {
        Batch storage batch = batches[batchId];
        require(!batch.finalized, "Already finalized");
        require(
            block.timestamp < batch.timestamp + CHALLENGE_PERIOD,
            "Challenge period ended"
        );

        // Верификация fraud proof
        if (verifyFraudProof(batchId, fraudProof)) {
            // Откат пакета
            delete batches[batchId];
            // Slash секвенсора
        }
    }

    // Финализация после периода оспаривания
    function finalizeBatch(uint256 batchId) external {
        Batch storage batch = batches[batchId];
        require(
            block.timestamp >= batch.timestamp + CHALLENGE_PERIOD,
            "Challenge period not ended"
        );
        batch.finalized = true;
    }
}
```

**Примеры:** Arbitrum, Optimism, Base

### ZK Rollups (Zero-Knowledge Rollups)

**ZK Rollups** используют криптографические доказательства (validity proofs) для подтверждения корректности транзакций.

**Как работают:**
1. Секвенсор собирает транзакции
2. Генерируется **ZK-proof** (доказательство с нулевым разглашением)
3. Proof отправляется на L1 вместе с данными
4. L1 контракт **верифицирует proof** — моментальная финальность

```
┌────────────────────────────────────────────┐
│              ZK Rollup                      │
│                                            │
│  Транзакции → ZK Prover → Validity Proof   │
│                                            │
└──────────────────┬─────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│              Ethereum L1                    │
│                                            │
│  Verifier Contract: verify(proof) = true   │
│         → State Update Applied              │
│                                            │
└────────────────────────────────────────────┘
```

**Примеры:** zkSync Era, StarkNet, Polygon zkEVM, Scroll, Linea

## Сравнение: Optimistic vs ZK Rollups

| Характеристика | Optimistic Rollups | ZK Rollups |
|----------------|-------------------|------------|
| **Безопасность** | Fraud proofs | Validity proofs |
| **Финальность** | ~7 дней | Мгновенная |
| **Вывод средств** | 7 дней (без моста) | Минуты |
| **Стоимость proof** | Низкая | Высокая (вычисления) |
| **EVM совместимость** | Полная | Сложнее (zkEVM) |
| **Зрелость** | Высокая | Развивается |
| **Gas на L1** | Меньше | Больше (proof) |

## Data Availability (Доступность данных)

**Data Availability** — критически важный аспект L2. Данные транзакций должны быть доступны для верификации.

### Варианты DA:

1. **On-chain DA (Ethereum)** — все данные хранятся в calldata на L1
   - Самый безопасный
   - Самый дорогой

2. **Validium** — данные хранятся off-chain (DAC)
   - Дешевле
   - Меньше безопасности

3. **Volition** — гибрид (выбор пользователя)
   - zkSync Era, StarkNet

4. **EIP-4844 (Proto-Danksharding)** — специальные "blobs" для L2
   - Значительно снижает стоимость
   - Данные доступны ~2 недели

```javascript
// Пример отправки данных с EIP-4844 blobs
// Транзакция типа 3 с blob данными
const tx = {
    type: 3, // Blob transaction
    to: rollupContract,
    data: batchCalldata,
    blobVersionedHashes: [blobHash], // Хэши blob данных
    maxFeePerBlobGas: 1000000000n,   // Max fee за blob gas
};
```

## Сравнительная таблица L2 решений

| L2 Solution | Тип | TPS | Время вывода | TVL (2024) | Особенности |
|-------------|-----|-----|--------------|------------|-------------|
| **Arbitrum One** | Optimistic | ~4,000 | 7 дней | $10B+ | Nitro, Stylus |
| **Optimism** | Optimistic | ~2,000 | 7 дней | $7B+ | OP Stack, Superchain |
| **Base** | Optimistic | ~2,000 | 7 дней | $5B+ | Coinbase, OP Stack |
| **zkSync Era** | ZK | ~2,000 | Часы | $1B+ | zkEVM, Account Abstraction |
| **StarkNet** | ZK | ~1,000 | Часы | $500M+ | Cairo, STARK proofs |
| **Polygon zkEVM** | ZK | ~2,000 | Часы | $100M+ | Type 2 zkEVM |
| **Scroll** | ZK | ~1,000 | Часы | $500M+ | Type 2 zkEVM |
| **Linea** | ZK | ~1,500 | Часы | $800M+ | ConsenSys |

## Преимущества L2

### 1. Масштабируемость
- Увеличение TPS в 10-100+ раз
- Ethereum может обслуживать миллионы пользователей

### 2. Низкие комиссии
```
Сравнение комиссий (swap на DEX):
├── Ethereum L1:     $5-50
├── Arbitrum:        $0.10-0.50
├── Optimism:        $0.10-0.50
├── zkSync Era:      $0.05-0.20
└── Base:            $0.01-0.10
```

### 3. Быстрые транзакции
- Подтверждение за секунды на L2
- Не нужно ждать блоков L1

### 4. Наследование безопасности Ethereum
- L2 полагается на консенсус Ethereum
- При проблемах можно выйти на L1

## Разработка на L2

Большинство L2 **EVM-совместимы**, что позволяет использовать стандартные инструменты:

```javascript
// Подключение к Arbitrum
const { ethers } = require('ethers');

// RPC endpoints
const providers = {
    arbitrum: new ethers.JsonRpcProvider('https://arb1.arbitrum.io/rpc'),
    optimism: new ethers.JsonRpcProvider('https://mainnet.optimism.io'),
    base: new ethers.JsonRpcProvider('https://mainnet.base.org'),
    zkSync: new ethers.JsonRpcProvider('https://mainnet.era.zksync.io'),
};

// Деплой контракта — идентичен Ethereum
async function deployOnL2(provider, wallet, contractFactory) {
    const signer = wallet.connect(provider);
    const contract = await contractFactory.connect(signer).deploy();
    await contract.waitForDeployment();
    return contract;
}
```

### Hardhat конфигурация для L2

```javascript
// hardhat.config.js
module.exports = {
    solidity: "0.8.20",
    networks: {
        arbitrum: {
            url: "https://arb1.arbitrum.io/rpc",
            chainId: 42161,
            accounts: [process.env.PRIVATE_KEY],
        },
        optimism: {
            url: "https://mainnet.optimism.io",
            chainId: 10,
            accounts: [process.env.PRIVATE_KEY],
        },
        base: {
            url: "https://mainnet.base.org",
            chainId: 8453,
            accounts: [process.env.PRIVATE_KEY],
        },
        zkSync: {
            url: "https://mainnet.era.zksync.io",
            chainId: 324,
            zksync: true, // Требует @matterlabs/hardhat-zksync
        },
    },
};
```

## Мосты (Bridges)

Для перемещения активов между L1 и L2 используются **мосты**:

```javascript
// Пример использования официального моста Arbitrum
const { L1ToL2MessageGasEstimator } = require('@arbitrum/sdk');

async function bridgeToArbitrum(amount) {
    const l1Provider = new ethers.JsonRpcProvider(L1_RPC);
    const l2Provider = new ethers.JsonRpcProvider(L2_RPC);

    // Оценка газа для L2
    const gasEstimator = new L1ToL2MessageGasEstimator(l2Provider);

    // Депозит ETH на L2
    const inbox = new ethers.Contract(INBOX_ADDRESS, INBOX_ABI, l1Signer);
    const tx = await inbox.depositEth({
        value: amount,
    });

    await tx.wait();
    console.log('Deposited to Arbitrum');
}
```

## Вывод

Layer 2 решения — это будущее масштабирования Ethereum и других блокчейнов. Они позволяют:
- Сохранить безопасность L1
- Значительно снизить комиссии
- Увеличить пропускную способность
- Обеспечить лучший UX

Выбор между Optimistic и ZK Rollups зависит от приоритетов:
- **Optimistic** — проще, дешевле, зрелее
- **ZK** — быстрее финальность, математическая гарантия

## Ресурсы

- [L2Beat](https://l2beat.com/) — аналитика L2 решений
- [Ethereum.org L2](https://ethereum.org/en/layer-2/) — официальная документация
- [Vitalik's Rollup Guide](https://vitalik.ca/general/2021/01/05/rollup.html) — статья Виталика о Rollups
