# StarkNet

## История создания

**StarkNet** - это permissionless ZK Rollup, разработанный компанией **StarkWare Industries**. Компания была основана в 2018 году командой криптографов и исследователей.

**Основатели:**
- **Eli Ben-Sasson** - криптограф, со-изобретатель STARK
- **Alessandro Chiesa** - криптограф, со-изобретатель SNARK
- **Uri Kolodny** - серийный предприниматель

**Ключевые даты:**
- **2018** - Основание StarkWare
- **2020** - Запуск StarkEx (кастомные ZK решения)
- **Ноябрь 2021** - Alpha на mainnet
- **Июль 2022** - StarkNet Token (STRK) анонсирован
- **2023** - Cairo 1.0 релиз
- **Февраль 2024** - Airdrop STRK

## Технология: ZK Rollup со STARK доказательствами

### Что такое STARK?

**STARK** (Scalable Transparent ARgument of Knowledge) - это криптографическая система доказательств:

```
┌─────────────────────────────────────────────────────────────┐
│                     STARK vs SNARK                          │
├────────────────────────┬────────────────────────────────────┤
│        STARK           │            SNARK                   │
├────────────────────────┼────────────────────────────────────┤
│ Transparent (no setup) │ Trusted setup required             │
│ Post-quantum secure    │ Vulnerable to quantum              │
│ Larger proofs (~200KB) │ Smaller proofs (~200B)            │
│ Faster proving         │ Slower proving                     │
│ Hash-based security    │ Elliptic curves                    │
└────────────────────────┴────────────────────────────────────┘
```

### Как работает ZK Rollup

```
┌──────────────────────────────────────────────────────────────┐
│                      StarkNet L2                              │
├──────────────────────────────────────────────────────────────┤
│  TX₁, TX₂, TX₃, ... TXₙ  →  Sequencer  →  Cairo VM          │
│                                              │                │
│                                              ▼                │
│                                     State Transition          │
│                                              │                │
│                                              ▼                │
│                                      STARK Prover             │
│                                              │                │
└──────────────────────────────────────────────│────────────────┘
                                               │ Proof + State diff
                                               ▼
┌──────────────────────────────────────────────────────────────┐
│                    Ethereum L1                                │
│              Verifier Contract validates proof                │
│                    State root updated                         │
└──────────────────────────────────────────────────────────────┘
```

**Преимущества:**
- Безопасность наследуется от Ethereum
- Сжатие: 1000+ транзакций в одном доказательстве
- Валидность гарантируется математикой

## Cairo - язык программирования

**Cairo** - это язык, специально разработанный для написания доказуемых программ.

### Cairo 1.0 (Sierra)

Cairo 1.0 - современная версия с Rust-подобным синтаксисом:

```rust
// Пример контракта на Cairo 1.0
#[starknet::interface]
trait ICounter<TContractState> {
    fn get_counter(self: @TContractState) -> u128;
    fn increase_counter(ref self: TContractState);
    fn decrease_counter(ref self: TContractState);
}

#[starknet::contract]
mod Counter {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        counter: u128,
    }

    #[constructor]
    fn constructor(ref self: ContractState, initial_value: u128) {
        self.counter.write(initial_value);
    }

    #[abi(embed_v0)]
    impl CounterImpl of super::ICounter<ContractState> {
        fn get_counter(self: @ContractState) -> u128 {
            self.counter.read()
        }

        fn increase_counter(ref self: ContractState) {
            let current = self.counter.read();
            self.counter.write(current + 1);
        }

        fn decrease_counter(ref self: ContractState) {
            let current = self.counter.read();
            self.counter.write(current - 1);
        }
    }
}
```

### Типы данных в Cairo

```rust
// Felts - основной тип (field elements)
let x: felt252 = 100;

// Целые числа
let a: u8 = 255;
let b: u16 = 65535;
let c: u32 = 4294967295;
let d: u64 = 18446744073709551615;
let e: u128 = 340282366920938463463374607431768211455;
let f: u256 = u256 { low: 1, high: 0 };

// Boolean
let flag: bool = true;

// Array
let mut arr: Array<felt252> = ArrayTrait::new();
arr.append(1);
arr.append(2);

// Struct
#[derive(Drop, Serde, starknet::Store)]
struct User {
    address: ContractAddress,
    balance: u256,
}
```

### Cairo VM (CVM)

```
┌─────────────────────────────────────────────────────────────┐
│                    Compilation Pipeline                      │
├─────────────────────────────────────────────────────────────┤
│  Cairo 1.0  →  Sierra  →  CASM  →  Execution Trace → STARK │
│  (Source)     (Safe IR)  (Assembly)     ↓                   │
│                                    Proof                     │
└─────────────────────────────────────────────────────────────┘
```

- **Sierra** - безопасный промежуточный код (Safe Intermediate Representation)
- **CASM** - Cairo Assembly, низкоуровневый код для CVM
- **Execution Trace** - запись всех шагов выполнения

## Account Abstraction

StarkNet имеет **нативную account abstraction** - все аккаунты являются смарт-контрактами:

```rust
#[starknet::interface]
trait IAccount<TContractState> {
    fn __validate__(self: @TContractState, calls: Array<Call>) -> felt252;
    fn __execute__(ref self: TContractState, calls: Array<Call>) -> Array<Span<felt252>>;
    fn is_valid_signature(self: @TContractState, hash: felt252, signature: Array<felt252>) -> felt252;
}

// Пример кастомной логики валидации
#[starknet::contract]
mod MultisigAccount {
    use starknet::storage::{StoragePointerReadAccess, Map, StorageMapReadAccess};

    #[storage]
    struct Storage {
        signers: Map::<ContractAddress, bool>,
        threshold: u8,
    }

    #[abi(embed_v0)]
    impl AccountImpl of super::IAccount<ContractState> {
        fn __validate__(self: @ContractState, calls: Array<Call>) -> felt252 {
            // Кастомная логика: проверка нескольких подписей
            let valid_signatures = self.count_valid_signatures();
            assert(valid_signatures >= self.threshold.read(), 'Not enough signatures');
            starknet::VALIDATED
        }

        fn __execute__(ref self: ContractState, calls: Array<Call>) -> Array<Span<felt252>> {
            // Выполнение вызовов
            self.execute_calls(calls)
        }

        fn is_valid_signature(
            self: @ContractState,
            hash: felt252,
            signature: Array<felt252>
        ) -> felt252 {
            // Проверка подписи
            'VALID'
        }
    }
}
```

**Возможности AA:**
- Мультиподпись из коробки
- Социальное восстановление
- Сессионные ключи
- Оплата газа в любом токене
- Пакетные транзакции

## Токеномика STRK

```
Общее предложение: 10,000,000,000 STRK

Распределение:
┌───────────────────────────────────────────┐
│ Community (50.1%)      │ 5,010,000,000   │
│ - Provisions          │ 900,000,000     │
│ - Future grants       │ 4,110,000,000   │
├───────────────────────────────────────────┤
│ Early Contributors    │ 3,240,000,000   │
│ (32.4%)               │                 │
├───────────────────────────────────────────┤
│ Investors (17.5%)     │ 1,750,000,000   │
└───────────────────────────────────────────┘

Использование STRK:
- Оплата комиссий (gas)
- Стейкинг для консенсуса
- Голосование в governance
```

## Инструменты разработки

### Scarb - Package Manager

```bash
# Установка
curl --proto '=https' --tlsv1.2 -sSf https://docs.swmansion.com/scarb/install.sh | sh

# Создание проекта
scarb new my_project

# Структура проекта
my_project/
├── Scarb.toml          # Манифест
├── src/
│   └── lib.cairo       # Точка входа
└── tests/              # Тесты
```

**Scarb.toml:**
```toml
[package]
name = "my_project"
version = "0.1.0"
cairo-version = "2.6.0"

[dependencies]
starknet = "2.6.0"

[[target.starknet-contract]]
sierra = true
```

### Starkli - CLI

```bash
# Установка
curl https://get.starkli.sh | sh
starkliup

# Создание кошелька
starkli signer keystore new ~/.starkli-wallets/keystore.json

# Account deployment
starkli account oz init ~/.starkli-wallets/account.json
starkli account deploy ~/.starkli-wallets/account.json

# Объявление контракта
starkli declare target/dev/my_project_Counter.contract_class.json

# Деплой контракта
starkli deploy <CLASS_HASH> <CONSTRUCTOR_ARGS>

# Вызов функции
starkli call <CONTRACT_ADDRESS> get_counter
starkli invoke <CONTRACT_ADDRESS> increase_counter
```

### Foundry for Starknet (snforge)

```bash
# Установка
curl -L https://raw.githubusercontent.com/foundry-rs/starknet-foundry/master/scripts/install.sh | sh

# Запуск тестов
snforge test
```

**Пример теста:**
```rust
#[cfg(test)]
mod tests {
    use snforge_std::{declare, ContractClassTrait};
    use super::{ICounterDispatcher, ICounterDispatcherTrait};

    #[test]
    fn test_counter_increase() {
        // Деплой контракта
        let contract = declare("Counter").unwrap();
        let (contract_address, _) = contract.deploy(@array![100]).unwrap();

        let dispatcher = ICounterDispatcher { contract_address };

        // Проверка начального значения
        assert(dispatcher.get_counter() == 100, 'Wrong initial value');

        // Увеличение и проверка
        dispatcher.increase_counter();
        assert(dispatcher.get_counter() == 101, 'Should be 101');
    }
}
```

## Экосистема

### Кошельки
- **Argent X** - ведущий кошелек с account abstraction
- **Braavos** - альтернатива с аппаратной безопасностью

### DeFi
- **JediSwap** - DEX
- **mySwap** - DEX
- **Ekubo** - концентрированная ликвидность
- **zkLend** - lending protocol
- **Nostra** - lending и stablecoin

### NFT
- **Unframed** - NFT маркетплейс
- **Element** - NFT платформа

### Инфраструктура
- **Voyager** - block explorer
- **Starkscan** - альтернативный explorer
- **Alchemy** - RPC провайдер
- **Infura** - RPC провайдер

## Архитектура контрактов

### Классы и инстансы

В StarkNet контракты разделены на **классы** и **инстансы**:

```
┌──────────────────┐
│   Contract Class │  ← Код (один раз)
│   (class_hash)   │
└────────┬─────────┘
         │ deploy
         ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Instance A       │  │ Instance B       │  │ Instance C       │
│ (address_A)      │  │ (address_B)      │  │ (address_C)      │
│ Storage: {...}   │  │ Storage: {...}   │  │ Storage: {...}   │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

**Преимущества:**
- Экономия на деплое (declare один раз)
- Возможность upgrade через замену class_hash

### Upgradeable Contracts

```rust
#[starknet::interface]
trait IUpgradeable<TContractState> {
    fn upgrade(ref self: TContractState, new_class_hash: ClassHash);
}

#[starknet::contract]
mod UpgradeableCounter {
    use starknet::ClassHash;
    use starknet::SyscallResultTrait;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        counter: u128,
        owner: ContractAddress,
    }

    #[abi(embed_v0)]
    impl UpgradeableImpl of super::IUpgradeable<ContractState> {
        fn upgrade(ref self: ContractState, new_class_hash: ClassHash) {
            // Проверка прав
            assert(get_caller_address() == self.owner.read(), 'Not owner');

            // Замена имплементации
            starknet::syscalls::replace_class_syscall(new_class_hash).unwrap_syscall();
        }
    }
}
```

## Преимущества и недостатки

### Преимущества

- **Безопасность L1** - наследует безопасность Ethereum
- **STARK доказательства** - пост-квантовая устойчивость, без trusted setup
- **Native Account Abstraction** - гибкость в дизайне аккаунтов
- **Cairo** - мощный язык для ZK программ
- **Масштабируемость** - тысячи транзакций в одном доказательстве
- **Низкие комиссии** - разделение стоимости между транзакциями

### Недостатки

- **Медленная финализация** - 1-2 часа до L1 подтверждения
- **Новый язык** - Cairo требует изучения
- **Меньше разработчиков** - по сравнению с Solidity
- **Централизованный секвенсор** - пока в процессе децентрализации
- **Сложность отладки** - ZK специфичные ошибки

## Производительность

```
Теоретический TPS:     2,000+ на L2
Реальный TPS:          100-500 (зависит от сложности)

Комиссия за транзакцию: ~$0.01-0.10 (зависит от газа на L1)
Время подтверждения L2: ~секунды
Время финализации L1:   ~1-2 часа

Сжатие данных:          ~100-1000x
```

## Roadmap

1. **Decentralized Sequencer** - переход к децентрализации
2. **Volition** - выбор между on-chain и off-chain data
3. **Recursive Proofs** - доказательства доказательств
4. **Applicative Recursion** - Cairo программы проверяют Cairo программы

## Ссылки

- [StarkNet Documentation](https://docs.starknet.io/)
- [Cairo Book](https://book.cairo-lang.org/)
- [StarkNet Foundry](https://foundry-rs.github.io/starknet-foundry/)
- [Voyager Explorer](https://voyager.online/)
- [StarkNet Ecosystem](https://www.starknet-ecosystem.com/)
