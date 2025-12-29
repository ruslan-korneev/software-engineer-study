# Альтернативные сети

## Зачем нужны альтернативные виртуальные машины?

Ethereum Virtual Machine (EVM) стала стандартом де-факто для смарт-контрактов, но имеет ряд ограничений:

1. **Производительность** - EVM не оптимизирована для высокой пропускной способности
2. **Дорогие вычисления** - сложные операции требуют много газа
3. **Ограниченная параллелизация** - транзакции выполняются последовательно
4. **Устаревший дизайн** - архитектура 2014-2015 года

Эти ограничения привели к появлению альтернативных подходов, каждый из которых решает свой набор проблем.

## Подходы к масштабируемости

### Layer 1 решения

| Сеть | Подход | TPS (теор.) | Консенсус |
|------|--------|-------------|-----------|
| Solana | Параллельная обработка + PoH | 65,000 | Tower BFT |
| Aptos | Move VM + Block-STM | 160,000 | AptosBFT |
| Sui | Объектно-ориентированная модель | 120,000 | Narwhal/Bullshark |
| Near | Шардинг (Nightshade) | 100,000+ | Doomslug |

### Layer 2 решения (ZK Rollups)

| Сеть | Технология | Доказательства | Язык |
|------|------------|----------------|------|
| StarkNet | Cairo VM | STARK | Cairo |
| zkSync Era | zkEVM | SNARK | Solidity |
| Polygon zkEVM | zkEVM | SNARK | Solidity |
| Scroll | zkEVM | SNARK | Solidity |

## Non-EVM экосистемы

### Solana Virtual Machine (SVM)

```
┌─────────────────────────────────────────┐
│              Solana Runtime              │
├─────────────────────────────────────────┤
│           BPF (Berkeley Packet Filter)   │
├─────────────────────────────────────────┤
│      Programs (Rust/C/C++)              │
├─────────────────────────────────────────┤
│      Account Model                       │
└─────────────────────────────────────────┘
```

**Особенности:**
- Программы компилируются в BPF байткод
- Параллельное выполнение транзакций (Sealevel)
- Модель аккаунтов вместо контрактов
- Программы stateless - данные хранятся в отдельных аккаунтах

### Cairo Virtual Machine (CVM)

```
┌─────────────────────────────────────────┐
│           Cairo Runner                   │
├─────────────────────────────────────────┤
│           CASM (Cairo Assembly)          │
├─────────────────────────────────────────┤
│           Cairo 1.0                      │
├─────────────────────────────────────────┤
│       STARK Prover/Verifier             │
└─────────────────────────────────────────┘
```

**Особенности:**
- Разработана специально для ZK-доказательств
- Детерминистичное выполнение без газа
- Native account abstraction
- Доказуемые вычисления (provable computation)

### Move Virtual Machine

```
┌─────────────────────────────────────────┐
│           Move Runtime                   │
├─────────────────────────────────────────┤
│           Move Bytecode                  │
├─────────────────────────────────────────┤
│           Move Language                  │
├─────────────────────────────────────────┤
│       Resource-oriented Model            │
└─────────────────────────────────────────┘
```

**Используется в:** Aptos, Sui

**Особенности:**
- Ресурсо-ориентированная модель
- Линейные типы (linear types)
- Формальная верификация

## Сравнение виртуальных машин

### Solana VM vs Cairo VM vs EVM

| Характеристика | EVM | Solana VM | Cairo VM |
|---------------|-----|-----------|----------|
| **Язык** | Solidity/Vyper | Rust/C | Cairo |
| **Модель данных** | Account-based | Account Model | Felts (поля) |
| **Параллелизм** | Нет | Да (Sealevel) | Да (off-chain) |
| **Доказательства** | Нет | Нет | STARK |
| **TPS** | ~15-30 | ~65,000 | ~2,000 (L2) |
| **Finality** | ~12 мин | ~400ms | ~1-2 часа* |
| **Газ** | Да | Compute Units | Steps |
| **Тьюринг-полнота** | Да | Да | Да |

*StarkNet finality зависит от публикации доказательства на L1

### Модели данных

```
EVM Account Model:
┌──────────────────┐
│ Contract Address │
├──────────────────┤
│ Balance          │
│ Nonce            │
│ Code             │
│ Storage (mapping)│
└──────────────────┘

Solana Account Model:
┌──────────────────┐      ┌──────────────────┐
│ Program Account  │      │ Data Account     │
├──────────────────┤      ├──────────────────┤
│ Executable: true │      │ Executable: false│
│ Owner: BPF Loader│      │ Owner: Program   │
│ Data: bytecode   │      │ Data: state      │
└──────────────────┘      └──────────────────┘

Cairo Account Model:
┌──────────────────┐
│ Contract Class   │──┐   ┌──────────────────┐
├──────────────────┤  └──>│ Contract Instance│
│ Class Hash       │      ├──────────────────┤
│ Sierra Code      │      │ Contract Address │
│ ABI              │      │ Storage          │
└──────────────────┘      │ Class Hash       │
                          └──────────────────┘
```

## Когда выбирать альтернативные сети?

### Solana подходит для:
- Высокочастотный трейдинг (DeFi)
- Игры с быстрыми транзакциями
- NFT маркетплейсы с низкими комиссиями
- Приложения, требующие низкую задержку

### StarkNet подходит для:
- Приложения с приоритетом безопасности
- Сложные вычисления (верифицируемые off-chain)
- Наследование безопасности Ethereum
- Account abstraction из коробки

### EVM-совместимые сети для:
- Портирование существующих dApps
- Использование готовых инструментов
- Большая экосистема разработчиков

## Тренды развития

1. **Modular Blockchain** - разделение execution, settlement, data availability
2. **Parallel Execution** - Monad, Sei, Aptos
3. **ZK Everywhere** - zkWASM, zkVM
4. **Cross-chain interoperability** - LayerZero, Wormhole

## Ссылки

- [Solana Documentation](https://docs.solana.com/)
- [StarkNet Documentation](https://docs.starknet.io/)
- [Move Language](https://move-language.github.io/move/)
- [L2Beat - Layer 2 Statistics](https://l2beat.com/)
